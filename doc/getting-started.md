# Getting Started

## Basics

Each element of Alda's syntax has a corresponding function in [alda.core].

For example, the following Alda score:

```
piano:
  c8 d e f g a b > c
```

...can be produced via alda-clj by using the corresponding functions in
alda.core:

```clojure
(require '[alda.core :refer :all])

(part "piano")
(note (pitch :c) (note-length 8))
(note (pitch :d))
(note (pitch :e))
(note (pitch :f))
(note (pitch :g))
(note (pitch :a))
(note (pitch :b))
(octave :up)
(note (pitch :c))
```

Assuming you have [Alda installed][install-alda] and the server is running
(`alda up`), you can play this score from your Clojure REPL by wrapping it in a
call to `alda.core/play!` and evaluating the form:

```clojure
(play!
  (part "piano")
  (note (pitch :c) (note-length 8))
  (note (pitch :d))
  (note (pitch :e))
  (note (pitch :f))
  (note (pitch :g))
  (note (pitch :a))
  (note (pitch :b))
  (octave :up)
  (note (pitch :c)))
```

The return value of the `play!` form is the string of Alda code that was
generated and sent to the Alda client:

```clojure
"piano:\nc8 d e f g a b > c"
```

[alda.core]: https://cljdoc.org/d/io.djy/alda-clj/CURRENT/api/alda.core
[install-alda]: https://github.com/alda-lang/alda#installation

You can always stop playback if things get out of hand:

```clojure
(stop!)
```

## Shelling out

You can conveniently issue arbitrary commands to the Alda client from the
comfort of your REPL:

```clojure
;; like running "alda version" in a terminal
(println (alda "version"))

;; like running "alda status" in a terminal
(println (alda "status"))
```

## History

Each time you successfully `play!` something, the generated Alda code is
appended to `alda.core/*alda-history*`, a string of Alda code representing the
score so far that is sent along for context on each call to the `alda` client.
This is what allows scores to be played incrementally, e.g.:

```clojure
;; conjure a guitar
(play!
  (part "guitar"))

;; play a few notes on the guitar
(play!
  (note (pitch :e) (note-length 8))
  (note (pitch :f :sharp))
  (note (pitch :g)))

;; play a few notes, still on the guitar
(play!
  (notes (pitch :a))
  (notes (pitch :b) (note-length 2)))
```

Between invocations of `play!`, the Alda client "remembers" which instrument(s)
were active and all of their properties (octave, volume, panning, etc.) so that
the context is not lost.

The history can be cleared at will:

```clojure
(clear-history!)
```

## Notes

The `note` function takes as arguments a pitch and (optionally) a duration.

```clojure
;; a D
(play!
  (note (pitch :d)))
;;=> "d"

;; an E eighth note
(play!
  (note (pitch :e) (note-length 8)))
;;=> "e8"

;; a C# whole note tied to a dotted half note
(play!
  (note (pitch :c :sharp)
        (duration (note-length 1)
                  (note-length 2 {:dots 1}))))
;;=> "c+1~2."
```

### Pitch

The `pitch` function can be used to construct a pitch out of a letter and any
number of accidentals (flats and sharps):

```clojure
(play!
  (note (pitch :b)))
;;=> "b"

(play!
  (note (pitch :b :flat)))
;;=> "b-"

(play!
  (note (pitch :f :sharp)))
;;=> "f+"

(play!
  (note (pitch :f :sharp :sharp)))
;;=> "f++"
```

### Duration

A simple example of a duration is a single note-length:

```clojure
;; a C half note
(play!
  (note (pitch :c) (note-length 2)))
;;=> "c2"
```

Alda also allows you to specify the length of a note in milliseconds:

```clojure
;; a C note lasting 423 ms
(play!
  (note (pitch :c) (ms 423)))
;;=> "c423ms"
```

Durations (whether they be `note-length` or `ms`) can be added together via the
`duration` function:

```clojure
;; A double-dotted quarter note, tied to an eighth note, tied to a note 456ms
;; long, tied to a "sixth" note. Try writing THAT in standard musical notation!
(play!
  (note (pitch :c)
        (duration (note-length 4 {:dots 2})
                  (note-length 8)
                  (ms 456)
                  (note-length 6))))
;;=> "c4..~8~456ms~6"
```

## Rests

The function for a rest in alda.core is called `pause`, so as not to conflict
with the super popular function `clojure.core/rest`:

```clojure
(play!
  (note (pitch :f) (note-length 4))
  (pause (note-length 4))
  (note (pitch :f) (note-length 4))
  (pause (note-length 4))
  (note (pitch :f) (note-length 4))
  (pause (note-length 4)))
;;=> "f4 r4 f4 r4 f4 r4"
```

## Chords

Wrap notes in a `chord` form to create a chord:

```clojure
(play!
  (chord
    (note (pitch :c) (note-length 1))
    (note (pitch :e :flat))
    (note (pitch :g))
    (note (pitch :b))))
;;=> "c1 / e- / g / b"
```

## Strings and sequences

Ultimately, all we're doing here is generating a string of Alda code and sending
it over to the Alda client.

Because each event just ends up being a string anyway, `play!` will happily
accept a string of Alda code in lieu of an event:

```clojure
(play!
  "piano: o4"
  "c16/e/g"
  (note (pitch :a))
  (note (pitch :b))
  "> c")
;;=> "piano: o4 c16/e/g a b > c"
```

For convenience, `play!` will also flatten the sequence of its arguments. This
means you can make liberal use of the handy sequence functions in Clojure's
standard library, without needing to worry about providing `play!` with a flat
sequence of events.

```clojure
(play!
  (part "piano")
  (for [t (repeatedly 4 #(+ 60 (rand-int 200)))]
    [(tempo t)
     (for [letter [:c :d :e :f :g]]
       (note (pitch letter) (note-length 8)))]))
;;=> "piano:\n(tempo 72) c8 d8 e8 f8 g8 (tempo 157) c8 d8 e8 f8 g8 (tempo 129) c8 d8 e8 f8 g8 (tempo 96) c8 d8 e8 f8 g8"
```

## Inline Lisp forms

The Alda language itself includes Lisp forms as a syntactic feature. This is
used to provide language features without needing to provide additional syntax.
For example, here is how you set the global tempo in an Alda score:

```
(tempo! 160)

piano: c8 d e f g
```

alda-clj allows code like this to be generated in a couple of different ways.
One way is that you can pass quoted S-expressions into `play!` and they will be
emitted verbatim into the string of generated Alda code:

```clojure
(play!
  '(tempo! 160)
  (part "piano")
  (note (pitch :c) (note-length 8))
  (note (pitch :d))
  (note (pitch :e))
  (note (pitch :f))
  (note (pitch :g)))
;;=> "(tempo! 160) piano:\nc8 d e f g"
```

Another way is that alda.core includes convenience functions for most (maybe
even all?) of the functions like `tempo!` that are available in the Alda Lisp
environment. Whenever alda.core includes one of these functions, you can simply
call it and it will have the same effect as emitting the S-expression directly
into the generated Alda code:

```clojure
(play!
  (tempo! 160)
  (part "piano")
  (note (pitch :c) (note-length 8))
  (note (pitch :d))
  (note (pitch :e))
  (note (pitch :f))
  (note (pitch :g)))
;;=> "(tempo! 160) piano:\nc8 d e f g"
```

## Other stuff

alda-clj supports every feature of the Alda language, including things not
covered here like [cram brackets][cram], [variables], and [markers].

The [API docs][api-docs] provide a reference of what functions are available in
alda.core and how they are used.

[cram]: https://cljdoc.org/d/io.djy/alda-clj/CURRENT/api/alda.core#cram
[variables]: https://cljdoc.org/d/io.djy/alda-clj/CURRENT/api/alda.core#set-variable
[markers]: https://cljdoc.org/d/io.djy/alda-clj/CURRENT/api/alda.core#marker
[api-docs]: https://cljdoc.org/d/io.djy/alda-clj/CURRENT/api/alda
