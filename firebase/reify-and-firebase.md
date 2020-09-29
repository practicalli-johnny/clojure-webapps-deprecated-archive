## reify and firebase
Discussion from Slack

https://clojurians.slack.com/archives/C053AK3F9/p1594065449151400

noob reify question:

(defn firebase-save [path v]
  (-> (FirebaseDatabase/getInstance)
      (.getReference path)
      (.setValueAsync
        (stringify-keys v)
        (reify DatabaseReference$CompletionListener
          (onComplete [err _ref] (throw err)))))

Goal is to have firebase throw the actual error when saving value (rather then a java.util.concurrent.Exception)I am trying to reify DatabaseRefrence$CompletionListener , but am getting the error;Can't define method not in interfaces: onCompleteAm not sure why clj is telling me ^ -- onComplete does seem to exist on the interface > https://firebase.google.com/docs/reference/android/com/google/firebase/database/DatabaseReference.CompletionListener(when i click into it from intellij, I see it exists too. How could I dig deeper? (edited)
ghadi  20:58
@stopa from the reify docstring:

    Note that the first parameter must be supplied to correspond to the target object ('this' in Java parlance) Thus

  methods for interfaces will take one more argument than do the
  interface declarations. (edited)
stopa  20:59
aah!
ghadi  20:59
the error is suggesting that there isn't a method with that exact signature
20:59
need [this err ref] as the args
20:59
(deftype, reify and defrecord all behave this same way)
stopa  21:00
Awesome, works like a charm, thanks @ghadi!
stopa  21:05
One more noob question:

(defn firebase-save [path v]
  (future
    (-> (FirebaseDatabase/getInstance)
        (.getReference path)
        (.setValue
          (stringify-keys v)
          (reify DatabaseReference$CompletionListener
            (onComplete [_this err ref]
              (if err
                ;; TODO future should throw?
                (throw err)
                ;; TODO future should complete?
                true)))))))

Goal: firebase gives me an onComplete handler. If there's an error, I want to throw the original error to the future, otherwise I want to complete itHere I would guess the future would complete without waiting for onCompleteHow could I do above?
noisesmith  21:07
you could create a promise, then put a deliver to that promise inside the onComplete method
21:07
that's cheap, and gives you an object that will block on deref (and return false for realized?) until that onComplete runs
21:07
after which it will return the value delivered in the onComplete on deref
21:08
if you want the future not to complete until the onComplete returns, you could put a deref of the promise as the last thing in the future
stopa  21:08

(defn firebase-save [path v]
  (future
    (let [p (promise)
          _work (-> (FirebaseDatabase/getInstance)
                    (.getReference path)
                    (.setValue
                      (stringify-keys v)
                      (reify DatabaseReference$CompletionListener
                        (onComplete [_this err ref]
                          (deliver p [err ref])))))
          [err ref] @p]
      (if err (throw err)
              ref))))

^ something like this?
noisesmith  21:08
that looks right, yeah
stopa  21:09
bam, thanks!
21:10
note: reason I am wrapping with future, rather then just returning promise, is so I can get the semantics, that by dereferencing an error, it would throw
21:10
^ is this idiomatic, or not quite?
noisesmith  21:12
you could get the same thing, without "wasting" a thread, by reifying IDeref
21:12
(using a promise to implement, throwing on some condition)
stopa  21:12
Oo interesting!
ghadi  21:14
you could also turn the completion notifs into data on a core.async channel
noisesmith  21:15

foo=> (def d (reify clojure.lang.IDeref (deref [this] (if (> (rand) 0.5)
 (throw (Exception. "you lose")) 2))))
#'foo/d
foo=> @d
Execution error at finops_admin.auth$reify__52601/deref (48914f44106185aac2a591f0a62c0
f52e50e2fdfe7863baaeea7e53dc016c825-init.clj:1).
you losefoo=> @d
2

stopa  21:17
Okay a bit of a noob question: How would I merge this reify Idea, with promise?Maybe I can do something like: have a promise that accepts expects a delivery of [err res]in IDeref, if err we throw
noisesmith  21:17
you would deref the promise and do the conditional in your original, inside the deref method
21:18
(reify IDeref (deref [this] (let [[err ref] @p] (if err (throw err) ref))) (edited)
21:18
or something similar
stopa  21:19
oo! Very cool. Okay let me try (hacking together)
noisesmith  21:19
since this is the beginners channel, I'll go into more detail - the deref method is what gets called when you use @, so here you are describing the behavior of this anonymous object if it is dereferenced
stopa  21:20

(defn throwable-promise [f]
  (let [p (promise)
        resolve #(deliver p [nil %])
        reject #(deliver p [% nil])
        throwable-p (reify IDeref
                      (deref [this]
                        (let [[err ref] @p]
                          (if err (throw err) ref)))]
    (f resolve reject)
    throwable-p))

noisesmith  21:22
oh, that's a nice way to encapsulate it
