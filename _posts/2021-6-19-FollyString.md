---
layout: post
title: Facebook's C++ String Class in Folly
---

In this blog post we will take an in depth look at how [Facebook's C++ String from Folly](https://github.com/facebook/folly/blob/master/folly/docs/FBString.md) works.
The post will also cover relevant core C++ concepts used to implement the string class.

## Post Outline
<!-- go over core string - how it is represented -->
<!-- basic fb string -> fbstring core -> small, medium, large -->


<!-- go over key string functions and compare them to std lib string -->
<!-- add (pushback), remove -->

<!-- concepts: -->
<!-- polymorphism - dynamic (with virtual functions) and static (using templates) -->
- [Intro](#intro)
- [Constructor](#constructor)
    - [Small Strings](#small-strings)
    - [Medium Strings](#medium-strings)
    - [Large Strings](#large-strings)
- [Destructor](#destructor)
- [Increasing Size of String](#increasing-size-of-string)
- [Decreasing Size of String](#decreasing-size-of-string)
- [Getting the String Data](#getting-the-string-data)
- [Playground Code](#playground-code)
- [Conclusion](#conclusion)
- [Resources](#resources)

## Intro
<!-- cover the string implementation in c++ used at facebook
see [here]() for the code's readme. -->


The idea behind folly's string class is to have three categories that strings may fall into: small, medium, and large.
Small strings are than or equal to 23 characters and are stored locally on the stack without memory allocation.
Medium strings are between 24 and 255 characters long and are stored in malloc allocated memory. 
They are also copied eagerly.
Large strings are greater than 255 characters, are stoed in malloc allocated memory, and copied lazily.

Let's go over general design of the string code.
Folly has its own basic_fbstring which is the std::basic_string equivalent that can be override by different kinds of character types.
The folly::basic_fbstring has a private data field of type Storage, which is the name given to the folly::fbstring_core.
The folly::fbstring_core is where the data storage and manipulation occurs.
<!-- basic fb string, fb string core -->
<!-- think that an actual fbstring would override the type with a char -->
<!-- ^yeah so look in FbStringTest.cc -> has stuff like 
  folly::basic_fbstring<char> stringMember; -->
```
template <
   typename E,
   class T = std::char_traits<E>,
   class A = std::allocator<E>,
   class Storage = fbstring_core<E>>
class basic_fbstring {
…
private:
 // Data
 Storage store_;
};
```

<!-- INCLUDE fbstring_core definition!!!!! with template -->

<!-- talk about structure -->
<!-- in storagethere are 3 strategies in the 2 bits of the rightmost char of the storage -->

## Constructor
<!-- constructor -->
Depending on the size of the string, we want to initialize the string differently.
Let's take a look at one of the constructors of fbstring_core below.
We can see that it the size variable determines which of `initSmall()`, `initMedium()`, and `initLarge()` get called.

```
  fbstring_core(
      const Char* const data,
      const size_t size,
      bool disableSSO = FBSTRING_DISABLE_SSO) {
    if (!disableSSO && size <= maxSmallSize) {
      initSmall(data, size);
    } else if (size <= maxMediumSize) {
      initMedium(data, size);
    } else {
      initLarge(data, size);
    }
    assert(this->size() == size);
    assert(size == 0 || memcmp(this->data(), data, size * sizeof(Char)) == 0);
  }
```

For large strings, the ml_ field

### Small Strings
The class folly::fbstring_core has three different structs that are used for each of the different sized strings.
There is a small anonymous union object used:

```
  union {
    uint8_t bytes_[sizeof(MediumLarge)]; // For accessing the last byte.
    Char small_[sizeof(MediumLarge) / sizeof(Char)];
    MediumLarge ml_;
  };
```

For small strings only the small_ character array is used to store the string.

Let's first go over the `initSmall()` function.
 
```
// Small strings are bitblitted
template <class Char>
inline void fbstring_core<Char>::initSmall(
    const Char* const data, const size_t size) {
  // Layout is: Char* data_, size_t size_, size_t capacity_
  static_assert(
      sizeof(*this) == sizeof(Char*) + 2 * sizeof(size_t),
      "fbstring has unexpected size");
  static_assert(
      sizeof(Char*) == sizeof(size_t), "fbstring size assumption violation");
  // sizeof(size_t) must be a power of 2
  static_assert(
      (sizeof(size_t) & (sizeof(size_t) - 1)) == 0,
      "fbstring size assumption violation");

// If data is aligned, use fast word-wise copying. Otherwise,
// use conservative memcpy.
// The word-wise path reads bytes which are outside the range of
// the string, and makes ASan unhappy, so we disable it when
// compiling with ASan.
#ifndef FOLLY_SANITIZE_ADDRESS
  if ((reinterpret_cast<size_t>(data) & (sizeof(size_t) - 1)) == 0) {
    const size_t byteSize = size * sizeof(Char);
    constexpr size_t wordWidth = sizeof(size_t);
    switch ((byteSize + wordWidth - 1) / wordWidth) { // Number of words.
      case 3:
        ml_.capacity_ = reinterpret_cast<const size_t*>(data)[2];
        FOLLY_FALLTHROUGH;
      case 2:
        ml_.size_ = reinterpret_cast<const size_t*>(data)[1];
        FOLLY_FALLTHROUGH;
      case 1:
        ml_.data_ = *reinterpret_cast<Char**>(const_cast<Char*>(data));
        FOLLY_FALLTHROUGH;
      case 0:
        break;
    }
  } else
#endif
  {
    if (size != 0) {
      fbstring_detail::podCopy(data, data + size, small_);
    }
  }
  setSmallSize(size);
}
```

The code either word wise copies of the data is aligned or a memcpy of Pod (plain old data).
Word width is 8 bytes and require either 0, 1, or 2 slots in an array.

```
template <class Pod>
inline void podCopy(const Pod* b, const Pod* e, Pod* d) {
  assert(b != nullptr);
  assert(e != nullptr);
  assert(d != nullptr);
  assert(e >= b);
  assert(d >= e || d + (e - b) <= b);
  memcpy(d, b, (e - b) * sizeof(Pod));
}
```

### Medium Strings
For medium sized strings, the MediumLarge field in the struct is used:

```
  struct MediumLarge {
    Char* data_;
    size_t size_;
    size_t capacity_;

    size_t capacity() const {
      return kIsLittleEndian ? capacity_ & capacityExtractMask : capacity_ >> 2;
    }

    void setCapacity(size_t cap, Category cat) {
      capacity_ = kIsLittleEndian
          ? cap | (static_cast<size_t>(cat) << kCategoryShift)
          : (cap << 2) | static_cast<size_t>(cat);
    }
  };
```

The ml_ field of the anonymous union points to the MediumLarge struct.
When a medium sized string is stored, the Char* data_ field points to a string allocated on the heap.

Medium sized strings are stored in malloc-allocated memory and copied eagerly.

```
template <class Char>
FOLLY_NOINLINE inline void fbstring_core<Char>::initMedium(
    const Char* const data, const size_t size) {
  // Medium strings are allocated normally. Don't forget to
  // allocate one extra Char for the terminating null.
  auto const allocSize = goodMallocSize((1 + size) * sizeof(Char));
  ml_.data_ = static_cast<Char*>(checkedMalloc(allocSize));
  if (FOLLY_LIKELY(size > 0)) {
    fbstring_detail::podCopy(data, data + size, ml_.data_);
  }
  ml_.size_ = size;
  ml_.setCapacity(allocSize / sizeof(Char) - 1, Category::isMedium);
  ml_.data_[size] = '\0';
}
```

### Large Strings

In the case of a large string, a RefCounted object is used.
The fields of the RefCounted struct are shown below.

```
  struct RefCounted {
    std::atomic<size_t> refCount_;
    Char data_[1];

    ...

  }
```

The ml_ data fields points to the data (that is copied on write).

```
template <class Char>
FOLLY_NOINLINE inline void fbstring_core<Char>::initLarge(
    const Char* const data, const size_t size) {
  // Large strings are allocated differently
  size_t effectiveCapacity = size;
  auto const newRC = RefCounted::create(data, &effectiveCapacity);
  ml_.data_ = newRC->data_;
  ml_.size_ = size;
  ml_.setCapacity(effectiveCapacity, Category::isLarge);
  ml_.data_[size] = '\0';
}
```

Below we can see the core RefCounted struct's create function. 

```
static RefCounted* create(size_t* size) {
    const size_t allocSize =
        goodMallocSize(getDataOffset() + (*size + 1) * sizeof(Char));
    auto result = static_cast<RefCounted*>(checkedMalloc(allocSize));
    result->refCount_.store(1, std::memory_order_release);
    *size = (allocSize - getDataOffset()) / sizeof(Char) - 1;
    return result;
}
```

It picks a good size to malloc and then increases the refcount by 1.
Then in initLarge, the data is actually set to point to the same location (although no copying has occurred yet).

<!-- what is the FOLLY_LIKELY annotation - preprocessor macro useful when the author bas better knowledge than the compiler of whether the branch condition is likely to take place - how to set this? set by writer of library -->


## Destructor
<!-- destructor -->
Now lets see how to destroy folly strings.
Let's take a look at the following destructor.

```
  ~fbstring_core() noexcept {
    if (category() == Category::isSmall) {
      return;
    }
    destroyMediumLarge();
  }
```

If the string is in the small category we can just return because the entire string is stored on the stack and will be freed when our fbstring goes out of scope.
Otherwise we need to call `destroyMediumLarge()`.

```
  FOLLY_NOINLINE void destroyMediumLarge() noexcept {
    auto const c = category();
    assert(c != Category::isSmall);
    if (c == Category::isMedium) {
      free(ml_.data_);
    } else {
      RefCounted::decrementRefs(ml_.data_);
    }
  }
```

If the string is in the medium category, the function calls free to free malloc allocated memory that is copied eagerly.
Otherwise the function decrements the ref count of the store.
If the reference count is 1 then free is called.

```
    static void decrementRefs(Char* p) {
      auto const dis = fromData(p);
      size_t oldcnt = dis->refCount_.fetch_sub(1, std::memory_order_acq_rel);
      assert(oldcnt > 0);
      if (oldcnt == 1) {
        free(dis);
      }
    }
```

<!-- folly no inline - don't want function to be inline (increases compile time and binary size) for some reason

why delete at one ref count? was in the fb string video

```
static void decrementRefs(Char* p) {
    auto const dis = fromData(p);
    size_t oldcnt = dis->refCount_.fetch_sub(1, std::memory_order_acq_rel);
    assert(oldcnt > 0);
    if (oldcnt == 1) {
        free(dis);
    }
}
``` -->

## Increasing Size of String

Let's take a look at the `push_back()` function, which appends a character c to the end of the string, resulting in an increase in size by 1.
We have the basic_fbstring which calls the `push_back()` function on its store_ member variable

```
  // basic_fbstring<E, T, A, Storage>
  void push_back(const value_type c) { // primitive
    store_.push_back(c);
  }
```

The function push_back sets the output of expandNoInit to be the character.
The output is a pointer to the newly allocated space that we dereference
and set to be the character.

```
  // fbstring_core<Char>
  void push_back(Char c) { *expandNoinit(1, /* expGrowth = */ true) = c; }
```

<!-- analyze how the below chunk of code works
think that this is the core of how the string expands, not sure what the noInit
means, think it means it already existed and not being initialized for the first time -->
<!-- oh wait its actually because its not initialized - the new region isn't initialized -->
<!-- caller should fill expanded area -->

<!-- ok so we got as input the delta which is the size to increase by -->

Let's analyze the function `expandNoinit()` (definition below).

```
  // definition
  Char* expandNoinit(
      const size_t delta,
      bool expGrowth = false,
      bool disableSSO = FBSTRING_DISABLE_SSO);
```

The general strategy is to first make room and then change size.
We first check to see if the string category is small.
If so then we call setSmallSize and return a pointer to the last location.

```
template <class Char>
inline Char* fbstring_core<Char>::expandNoinit(
    const size_t delta,
    bool expGrowth, /* = false */
    bool disableSSO /* = FBSTRING_DISABLE_SSO */) {
  // Strategy is simple: make room, then change size
  assert(capacity() >= size());
  size_t sz, newSz;
  if (category() == Category::isSmall) {
    sz = smallSize();
    newSz = sz + delta;
    if (!disableSSO && FOLLY_LIKELY(newSz <= maxSmallSize)) {
      setSmallSize(newSz);
      return small_ + sz;
    }
    reserveSmall(
        expGrowth ? std::max(newSz, 2 * maxSmallSize) : newSz, disableSSO);
  } else {
    sz = ml_.size_;
    newSz = sz + delta;
    if (FOLLY_UNLIKELY(newSz > capacity())) {
      // ensures not shared
      reserve(expGrowth ? std::max(newSz, 1 + capacity() * 3 / 2) : newSz);
    }
  }
  assert(capacity() >= newSz);
  // Category can't be small - we took care of that above
  assert(category() == Category::isMedium || category() == Category::isLarge);
  ml_.size_ = newSz;
  ml_.data_[newSz] = '\0';
  assert(size() == newSz);
  return ml_.data_ + sz;
}
```


Otherwise we are not using small strings and instead call reserveSmall.
The function reserveSmall is branched into small, medium, large to handle cases where the new string size is increased into medium and large categories.

```
template <class Char>
FOLLY_NOINLINE inline void fbstring_core<Char>::reserveSmall(
    size_t minCapacity, const bool disableSSO) {
  assert(category() == Category::isSmall);
  if (!disableSSO && minCapacity <= maxSmallSize) {
    // small
    // Nothing to do, everything stays put
  } else if (minCapacity <= maxMediumSize) {
    // medium
    // Don't forget to allocate one extra Char for the terminating null
    auto const allocSizeBytes =
        goodMallocSize((1 + minCapacity) * sizeof(Char));
    auto const pData = static_cast<Char*>(checkedMalloc(allocSizeBytes));
    auto const size = smallSize();
    // Also copies terminator.
    fbstring_detail::podCopy(small_, small_ + size + 1, pData);
    ml_.data_ = pData;
    ml_.size_ = size;
    ml_.setCapacity(allocSizeBytes / sizeof(Char) - 1, Category::isMedium);
  } else {
    // large
    auto const newRC = RefCounted::create(&minCapacity);
    auto const size = smallSize();
    // Also copies terminator.
    fbstring_detail::podCopy(small_, small_ + size + 1, newRC->data_);
    ml_.data_ = newRC->data_;
    ml_.size_ = size;
    ml_.setCapacity(minCapacity, Category::isLarge);
    assert(capacity() >= minCapacity);
  }
}
```

<!-- medium case !!!!!!!!-  -->
<!-- performs a malloc and then memcpy's over the small data using podCopy-->

In the medium and large cases we call reserve which branches into `reserveMedium()` and `reserveLarge()`.

```
  FOLLY_NOINLINE
  void reserve(size_t minCapacity, bool disableSSO = FBSTRING_DISABLE_SSO) {
    switch (category()) {
      case Category::isSmall:
        reserveSmall(minCapacity, disableSSO);
        break;
      case Category::isMedium:
        reserveMedium(minCapacity);
        break;
      case Category::isLarge:
        reserveLarge(minCapacity);
        break;
      default:
        folly::assume_unreachable();
    }
    assert(capacity() >= minCapacity);
  }
```

In `reserveMedium()`, if there isn't enough capacity then we memcpy over the small data (using `podCopy()`).
If we need to convert into a large string, then we recursively call reserve, which will call `reserveLarge()`.

```
template <class Char>
FOLLY_NOINLINE inline void fbstring_core<Char>::reserveMedium(
    const size_t minCapacity) {
  assert(category() == Category::isMedium);
  // String is not shared
  if (minCapacity <= ml_.capacity()) {
    return; // nothing to do, there's enough room
  }
  if (minCapacity <= maxMediumSize) {
    // Keep the string at medium size. Don't forget to allocate
    // one extra Char for the terminating null.
    size_t capacityBytes = goodMallocSize((1 + minCapacity) * sizeof(Char));
    // Also copies terminator.
    ml_.data_ = static_cast<Char*>(smartRealloc(
        ml_.data_,
        (ml_.size_ + 1) * sizeof(Char),
        (ml_.capacity() + 1) * sizeof(Char),
        capacityBytes));
    ml_.setCapacity(capacityBytes / sizeof(Char) - 1, Category::isMedium);
  } else {
    // Conversion from medium to large string
    fbstring_core nascent;
    // Will recurse to another branch of this function
    nascent.reserve(minCapacity);
    nascent.ml_.size_ = ml_.size_;
    // Also copies terminator.
    fbstring_detail::podCopy(
        ml_.data_, ml_.data_ + ml_.size_ + 1, nascent.ml_.data_);
    nascent.swap(*this);
    assert(capacity() >= minCapacity);
  }
}
```

<!-- reserveLarge -->
<!-- call RefCounted::realocate -->
<!-- calls goodMallocSize which calls nallocx (gets size) and then smart realloc which is just memcpy/malloc -->
For `reserveLarge()` if we call `RefCounted::reallocate` which essentially performs a memcpy and a malloc.

```
template <class Char>
FOLLY_NOINLINE inline void fbstring_core<Char>::reserveLarge(
    size_t minCapacity) {
  assert(category() == Category::isLarge);
  if (RefCounted::refs(ml_.data_) > 1) { // Ensure unique
    // We must make it unique regardless; in-place reallocation is
    // useless if the string is shared. In order to not surprise
    // people, reserve the new block at current capacity or
    // more. That way, a string's capacity never shrinks after a
    // call to reserve.
    unshare(minCapacity);
  } else {
    // String is not shared, so let's try to realloc (if needed)
    if (minCapacity > ml_.capacity()) {
      // Asking for more memory
      auto const newRC = RefCounted::reallocate(
          ml_.data_, ml_.size_, ml_.capacity(), &minCapacity);
      ml_.data_ = newRC->data_;
      ml_.setCapacity(minCapacity, Category::isLarge);
    }
    assert(capacity() >= minCapacity);
  }
}
```

Note that in `expandNoinit()`, `reserveLarge()` has a FOLLY_UNLIKELY macro around it.
What that does is tell the CPU that it is unlikely that this branch will be executed.
FOLLY_UNLIKELY is a wrapper around GCC's built in expect which provides the compiler with branch prediction information.

```
#if __GNUC__
#define FOLLY_DETAIL_BUILTIN_EXPECT(b, t) (__builtin_expect(b, t))
#else
#define FOLLY_DETAIL_BUILTIN_EXPECT(b, t) b
#endif
```

<!-- if likely then what!?!? what to do when increased capacity? -->
<!-- >capacity() -> size_t capacity() const { return backend_.capacity(); }
 size_t capacity() const {
      return kIsLittleEndian ? capacity_ & capacityExtractMask : capacity_ >> 2;
    } -->
<!-- ok so think just read as if the FOLLY_UNLIKELY isn't there -->
<!-- apparently its a user defined thing - by user i mean the library -->
<!-- looks like its a wrapper around __builtin_expect -->

<!-- what is built in expect - its a gnuc specific thing -->
<!-- provides compiler with branch prediction information -->
<!-- giving the gnu compiler this branch prediction information can let the resulting assembly instructions be arranged so that we can  -->


<!-- based off of example taken from gnu website -->
<!-- below example says that we do not expect x to be 0 -->
<!-- the default value that the compiler which expect that the builtin expect to be true is 90% and this can be modified -->

Below is an example of builtin expect taken from the gnu website (link in resources).
It says that we do not expect x to be 0 and therefore the compiler will expect this to be true 90% of the time (the percentage can be changed).

```
if (__builtin_expect (x, 0))
  foo ();
```

Note that built in expect is only a gcc construct to another compiler like clang won't know what it is, hence the #if header guard.

<!-- either way we update the size and set the last value to be null terminated -->


<!-- ------------------------------- -->
<!-- large case !!!!!!!!!!!-->
<!-- creates a new refcounted struct -->
<!-- performs memcpy and sets the medium large data corresponding structs -->

<!-- if we are doing a medium or large string, then we check to see if the new size is greater than the capacity -->
<!-- if its unlikely, then we call reserve -->

<!-- GO OVER reserveMedium and reserveLarge!!!!!!!!!!!!! -->


<!-- reserveMedium - if within capacity do nothing, if fit in medium size, call smartRealloc -->
<!-- tries to reallocate a buffer of which only first currentSize bytes are used -->
<!-- basically just a slightly more complicated memcpy and malloc -->
<!-- otherwise need to convert from medium to large string -->
<!-- in this case - looks like called reserve recurisvely - probably to call reserveLarge -->

## Decreasing Size of String
<!-- thinking of analyzing either erase or pop_back -->
<!-- erase deletes part of the string, reducing its length -->
<!-- pop back just removes the last character, think erase will be more interesting -->

<!-- ok so lets take a look at erase(...) -->
<!-- the key function signature is the pos = 0,n = npos -->
<!-- the other 2 erase functions call this function -->

```
  basic_fbstring& erase(size_type pos = 0, size_type n = npos) {
    Invariant checker(*this);

    enforce<std::out_of_range>(pos <= length(), "");
    procrustes(n, length() - pos);
    std::copy(begin() + pos + n, end(), begin() + pos);
    resize(length() - n);
    return *this;
  }
```

<!-- at the core this function error checks the input and makes sure that the parameters are valid  -->
<!-- then it copies the part of the string after the removed part (if any) 
up into where the erased part of the string originally was, effectively
cutting out the desired part of the string

then the string is resized by calling the resize function
-->

<!-- the resize function branches into either one of 2 categories-->
<!-- the first calls the shrink -->
<!-- the second branch calls expandNoinit as descsueed above and then initialized the newly allocated space using  -->
```
template <typename E, class T, class A, class S>
inline void basic_fbstring<E, T, A, S>::resize(
    const size_type n, const value_type c /*= value_type()*/) {
  Invariant checker(*this);

  auto size = this->size();
  if (n <= size) {
    store_.shrink(size - n);
  } else {
    auto const delta = n - size;
    auto pData = store_.expandNoinit(delta);
    fbstring_detail::podFill(pData, pData + delta, c);
  }
  assert(this->size() == n);
}
```

```
  void shrink(const size_t delta) {
    if (category() == Category::isSmall) {
      shrinkSmall(delta);
    } else if (
        category() == Category::isMedium || RefCounted::refs(ml_.data_) == 1) {
      shrinkMedium(delta);
    } else {
      shrinkLarge(delta);
    }
  }
```

below is the shrink size for the small

```
template <class Char>
inline void fbstring_core<Char>::shrinkSmall(const size_t delta) {
  // Check for underflow
  assert(delta <= smallSize());
  setSmallSize(smallSize() - delta);
}
```
it basically calls setSmall size which: null terminates the string at the new size

```
  void setSmallSize(size_t s) {
    // Warning: this should work with uninitialized strings too,
    // so don't assume anything about the previous value of
    // small_[maxSmallSize].
    assert(s <= maxSmallSize);
    constexpr auto shift = kIsLittleEndian ? 0 : 2;
    small_[maxSmallSize] = char((maxSmallSize - s) << shift);
    small_[s] = '\0';
    assert(category() == Category::isSmall && size() == s);
  }
```

shrink medium updates the mediumlarge struct's size field and null terminates the data
```
template <class Char>
inline void fbstring_core<Char>::shrinkMedium(const size_t delta) {
  // Medium strings and unique large strings need no special
  // handling.
  assert(ml_.size_ >= delta);
  ml_.size_ -= delta;
  ml_.data_[ml_.size_] = '\0';
}
```

shrink large 
```
template <class Char>
inline void fbstring_core<Char>::shrinkLarge(const size_t delta) {
  assert(ml_.size_ >= delta);
  // Shared large string, must make unique. This is because of the
  // durn terminator must be written, which may trample the shared
  // data.
  if (delta) {
    fbstring_core(ml_.data_, ml_.size_ - delta).swap(*this);
  }
  // No need to write the terminator.
}
```

it calls swap as defined below which basically swaps the newly created fbstring_core's ml field with the current ml's field
```
  void swap(fbstring_core& rhs) {
    auto const t = ml_;
    ml_ = rhs.ml_;
    rhs.ml_ = t;
  }
```

## Getting the String Data
<!-- think this is where could see the lazy vs eagerly for large vs medium sized strings to come into play -->
<!-- use the at function -->

Getting the string data is very simple and simply involves calling the at function which uses the operator[].

```
  const_reference at(size_type n) const {
    enforce<std::out_of_range>(n < size(), "");
    return (*this)[n];
  }
```

## Playground Code
<!-- try out using the string class - look at their string tests to see how to test this -->
<!-- try out different sizes -->

<!-- how to specify the FOLLY_LIKELY/UNLIKELY macros? -->

## Conclusion

## Resources
- [GCC/GNU built in expect](https://gcc.gnu.org/onlinedocs/gcc/Other-Builtins.html)
- [Folly Github](https://github.com/facebook/folly/blob/master/folly/docs/FBString.md)