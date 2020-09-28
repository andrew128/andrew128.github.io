---
layout: post
title: Balls and Bins Proof (9/28/20)
---
A full walk through of the balls and bins proof, split up into 4 parts.

## Post Outline
- [Intro](#intro)
- [Part 1: Upper Bound of > k balls in a given bin](#part-1)
- [Part 2: Upper Bound of > k balls in any bin](#part-2)
- [Part 3: Finding the right k](#part-3)
- [Part 4: Conclusion](#part-4)
- [Resources](#resources)

## Intro
In the balls and bins problem, we want to show that if $n$ balls are throw independently and uniformly into $n$ bins, the max bin size will be $O(\frac{\log n}{\log \log n})$ with high probability (specifically $1 - \frac{1}{n}$).
An example of where this can be applied is in hash tables, where we assume that we want to store $n$ values inside a hash table of size $n$.
We can then say with high probability that the longest get or put operation will be on the order of $O(\frac{\log n}{\log \log n})$.

## Part 1

Let's start by coming up with an expression for the probability that a given bin out of the $n$ total bins contains exactly $k$ balls of the $n$ total balls.
We will use the notation $X_{=k}$ to denote this event.
The probability that a given ball lands in a given bin is then $\frac{1}{n}$.
This is because we assume the balls are distributed uniformly.
For a given subset of the $n$ total balls of size $k$, the probability that all of those balls in that specific subset land in a given bin is then $\frac{1}{n^k}$.
The total number of subsets of size $k$ from a total set of size $n$ is ${n \choose k}$.
Therefore, our total probability that there are a subset of $k$ balls in a particular bin is:
$${n \choose k} \frac{1}{n^k}$$

However, we want exactly $k$ balls, which means we also need to factor in the probability that the rest of the balls do not make it into the particular bin.
We can apply the reverse logic.
The probability that a given ball does not fall into a given bin is $1 - \frac{1}{n}$.
We have $n - k$ balls left after our original $k$ balls have already made it into the bin.
Then, the probability that the rest of the $n - k$ balls do not make it into the bin is:

$$(1 - \frac{1}{n})^{n - k}$$

We can combine our previous 2 expressions to get the expression for the probability that exactly $k$ balls land in a given bin.

$$P(X_{=k}) = {n \choose k} \frac{1}{n^k}(1 - \frac{1}{n})^{n - k}$$

If we think about it, the expression ${n \choose k} \frac{1}{n^k}$ is actually an upper bound on the probability that there are more than $k$ balls that fall into a given bin, denoted by $P(X_{> k})$.
The term only includes the probability that a given $k$ balls will fall into the bin and does not say anything about the remaining balls.
The only difference between the below expression for $P(X_{> k})$ and the above expression for $P(X_{= k})$ is the $(1 - \frac{1}{n})^{n - k}$.
This term $(1 - \frac{1}{n})^{n - k}$ represents the added restriction that none of the other $n - k$ balls can fall into the bin.
Removing it means the other $n - k$ balls may or may not fall into the bin.
If we were to add any more restrictions that more than $k$ would have to be in the bin, we would be adding terms that are smaller than $1$ because they are probabilities, and make the resulting expression less than the below expression.

As a side note, we could have just presented the expression below without first deriving the expression for $P(X_{=k})$ but I think the expression for $P(X_{=k})$ helps to add context.

$$P(X_{> k}) \leq {n \choose k} \frac{1}{n^k}$$

## Part 2

We will now expand the combination term out of the above expression.

$$P(X_{> k}) \leq {n \choose k} \frac{1}{n^k}$$

$$ \leq \frac{(n)(n-1)...(n-k+1)}{k!}\frac{1}{n^k}$$

$$ \leq \frac{(n)(n-1)...(n-k+1)}{k(k-1)...(1)}\frac{1}{n^k}$$

$$ \leq (\frac{n}{k})^k\frac{1}{n^k}$$

$$ \leq (\frac{n}{k})^k\frac{1}{k!}$$

$$ \leq \frac{1}{k!}$$

Using union bound (aka summing across all the different possible bins), we have that for any bin, the probability that the number of balls is bigger than $k$ is (a stands for any):

$$P(X_{a, > k}) \leq \frac{n}{k!}$$

## Part 3

To prove our high probability bound of $1 - \frac{1}{n}$, we want to choose a value for $k$ such that $\frac{n}{k!} \approx \frac{1}{n}$.
Let's first rearrange the expression and then take the log of both sides:

$$\frac{n}{k!} \approx \frac{1}{n}$$

$$n^2 \approx k!$$

$$\ln(k!) \approx 2\ln(n)$$

$$\ln(k!) \approx \ln(n)$$

We will now use Stirling's approximation for factorials for the left side of the expression:
$$\ln(k!) = k\ln k - k \approx \ln(n)$$

$$k(\ln k - 1) \approx \ln(n)$$

$$k \approx \frac{\ln(n)}{\ln k - 1}$$

$$\approx \frac{\ln(n)}{\ln k - 1}$$

So because $k \approx \frac{\ln(n)}{\ln k - 1}$, we can sub it back into the above expression and get:

$$\approx \frac{\ln(n)}{\ln \frac{\ln(n)}{\ln k - 1} - 1}$$

$$\approx \frac{\ln(n)}{\ln \ln n - \ln\ln (k - 1) - 1}$$

As $n$ gets large, we can disregard the constants and the expressions with $k$ to get:

$$k \approx \frac{\ln(n)}{\ln \ln n}$$

## Part 4

We can now rewrite the below expression using our newly found approximation of $k$ such that $\frac{n}{k!} \approx \frac{1}{n}$ for any bucket.

$$P(X_{a, > k}) \leq \frac{n}{k!}$$

$$P(X_a > \frac{\ln(n)}{\ln \ln n}) < \frac{1}{n}$$

The above expression can be interpreted as: the probability that any bucket has more than $\frac{\ln(n)}{\ln \ln n}$ balls is less than $\frac{1}{n}$.
In other words, we will have at most $\frac{\ln(n)}{\ln \ln n}$ balls max in any bucket with probability $1 - \frac{1}{n}$.

## Resources
- [Toronto](http://www.cs.toronto.edu/~toni/Courses/263-2013/handouts/hashing-uiuc.pdf)
- [Purdue](https://www.cs.purdue.edu/homes/hmaji/teaching/Spring%202017/lectures/03.pdf)
- [Princeton](https://www.cs.princeton.edu/courses/archive/fall16/cos521/Lectures/lec1.pdf)