# Implementing a Throttler

## In purely functional Scala

---

## What is a Throttler?

* server-side <!-- .element: class="fragment"-->
* accept at a given rate <!-- .element: class="fragment"-->
* reject above the rate <!-- .element: class="fragment"-->

---

## What is "Pure Functional Programming"?

* referential transparency <!-- .element: class="fragment"-->
* same inputs = same outputs <!-- .element: class="fragment"-->
* IO Monad <!-- .element: class="fragment"-->

Note: from this all FP complexity arises

---

## How to throttle?

Note: does anyone know an alg?

---

## Token/Leaky Bucket

<!-- .slide: data-background="./leaky.jpg" -->

---

## Concurrency

* Optimistic <!-- .element: class="fragment"-->
* java.util.concurrent.atomic.AtomicReference <!-- .element: class="fragment"-->
* compareAndSet <!-- .element: class="fragment"-->
* Monotonic clock <!-- .element: class="fragment"-->

Note: Optimistic as opposed to lock-based

---

## Implementation

```
create bucket with bucketCapacity

new threadlike {
  every updateInterval {
    if room, add token to bucket
  }
}

takeToken(updateInterval) {
  remove token from bucket
}
```

Note: Good approach, many open source like this

---

## A better implementation

```
create bucket with bucketCapacity

takeToken(updateInterval) {
  timeDifference = currentTime - lastUpdateTime
  tokensToAdd = timeDifference / updateInterval
  newTotal = min(previousTokens + tokensToAdd, bucketCapacity)
  if newTotal > 1 {
    set total tokens to newTotal - 1
    return token
  } else {
    set total tokens to newTotal
  }
}
```

---

## And in Scala

```scala
override def takeToken: F[TokenAvailability] = {
  val attemptUpdate = counter.access.flatMap {
    case ((previousTokens, previousTime), setter) =>
      getTime.flatMap(currentTime => {
        val timeDifference = currentTime - previousTime
        val tokensToAdd = timeDifference.toDouble / refillEvery.toNanos.toDouble
        val newTokenTotal = Math.min(previousTokens + tokensToAdd, capacity.toDouble)

        val attemptSet: F[Option[TokenAvailability]] = if (newTokenTotal >= 1) {
          setter((newTokenTotal - 1, currentTime))
            .map(_.guard[Option].as(TokenAvailable))
        } else {
          val timeToNextToken = refillEvery.toNanos - timeDifference
          val successResponse = TokenUnavailable(timeToNextToken.nanos.some)
          setter((newTokenTotal, currentTime)).map(_.guard[Option].as(successResponse))
        }

        attemptSet
      })
  }

  def loop: F[TokenAvailability] = attemptUpdate.flatMap { attempt =>
    attempt.fold(loop)(token => token.pure[F])
  }
  loop
}
```

---

# Questions?
