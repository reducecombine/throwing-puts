# throwing-puts

`throwing-puts` consists of `>!` and `>!!` equivalents which throw an exception if the channel cannot be written to
(i.e. it would cause a block/park).

Throwing exceptions instead of blocking (possibly forever) is occasionally desirable for "fail-fast" detection/handling
of truly exceptional circumstances.

Note that I don't recommend an extensive use of exception-based designs. Generally blocking/parking is a desirable
property of async systems, it is at certain system boundaries where it becomes a "this should never happen" thing.

## Sample use case

Let's say you have a background job system, which for reliability, first persists its messages to Redis, SQS, or such. But,
the enqueueing itself (presumably done through core.async) can fail:

```clojure
(go (>! redis-enqueuer some-message)) ;; can eventually block forever if SQS/Redis connectivity is down
```

What should you do in such circumstances?

* Blocking forever would hide the issue and degrade performance
* Using a dropping-buffer/sliding-buffer would cause message loss and does nothing to help addressing the underlying problem
* Using a `(chan 1000000)` would uselessly accumulate messages, risking eventual loss

While surely there's a variety of sophisticated choices to tackle this scenario, a simple one seems to throw exceptions.

* If thrown outside `go` blocks, such thrown exceptions abort the whole operation (such as a HTTP request/response cycle), and are
reported/logged properly through your (presumed) configuration
* If thrown inside `go` blocks, exceptions are less robust, but one still gets the presumed error reporting (which includes the
lost message itself; so one could eventually recover and re-enqueue it).

## Usage

[Clojars](https://clojars.org/throwing-puts)

I recommend `throwing/>!` (with that prefix or an analog one) so you can cleanly distinguish it from core.async `>!`.

```clojure
(require '[throwing-puts.core :as throwing])

;; >!! has the same signature and general behavior than its core.async counterpart.
;; If the write wasn't performed before #'throwing-puts.core/default-timeout-ms milliseconds, it will throw an ExceptionInfo.
(throwing/>!! (chan) 42)

;; There's >! too, slightly less recommendable (errors won't pop up).
;; Make sure to `setDefaultUncaughtExceptionHandler` accordingly
(go
  (throwing/>! (chan) 42))

(throwing/>!! (chan) 42 400) ;; Alternative timeout of 400 ms

(binding [throwing/*default-timeout-ms* 100] ;; another alternative, apt to be set in middleware, etc
  (throwing/>!! (chan) 42))
```

## One thing well

This is not an "async utils" library, and you can trust that that will always be the case.

## License

Copyright © 2018 vemv.net

Distributed under the Eclipse Public License either version 1.0 or (at your option) any later version.
