# Integrant Advanced Features

## Contents
- Derived keywords
- Composite keys
- Composite references
- Annotations
- load-namespaces
- load-hierarchy and load-annotations
- assert-key
- normalize-key

---

## Derived Keywords

Clojure's keyword hierarchy lets keys refer to their descendants. Set up a hierarchy with `derive`:

```clojure
(derive :adapter/jetty :adapter/ring)
```

Now `:adapter/ring` can be used wherever `:adapter/jetty` would be used:

```clojure
;; Init only ring adapters (matches :adapter/jetty because it is derived from :adapter/ring)
(ig/init config [:adapter/ring])
```

Refs can also use parent keywords, as long as the ref is **unambiguous** (only one matching key in the configuration):

```edn
{:handler/greet {:port #ig/ref :adapter/ring}}   ; fine if only one :adapter/ring descendant
```

If the ref is ambiguous (two keys both derive from `:adapter/ring`), Integrant throws. Use a more specific key or a composite reference.

---

## Composite Keys

Sometimes you need two instances of the same type of key (e.g. two web servers on different ports). Use a **vector of qualified keywords** as the key:

```edn
{[:adapter/jetty :example/web-1] {:port 8080, :handler #ig/ref :handler/greet}
 [:adapter/jetty :example/web-2] {:port 8081, :handler #ig/ref :handler/greet}
 :handler/greet {:name "Alice"}}
```

Integrant treats a composite key as if it were derived from every keyword in the vector. The equivalent `derive` approach:

```clojure
(derive :example/web-1 :adapter/jetty)
(derive :example/web-2 :adapter/jetty)
```

The composite key syntax is more concise and keeps the configuration self-contained.

---

## Composite References

A **composite reference** (vector of keywords inside `#ig/ref` or `#ig/refset`) matches keys derived from **every** keyword in the vector simultaneously. Use this to scope refs to a group:

```edn
{[:group/a :adapter/jetty] {:port 8080, :handler #ig/ref [:group/a :handler/greet]}
 [:group/a :handler/greet] {:name "Alice"}
 [:group/b :adapter/jetty] {:port 8081, :handler #ig/ref [:group/b :handler/greet]}
 [:group/b :handler/greet] {:name "Bob"}}
```

Each `[:group/a :adapter/jetty]` key only resolves the `[:group/a :handler/greet]` ref, not the group B handler. This enables multiple independent "groups" in one configuration without key collisions.

---

## Annotations

Attach documentation or metadata to a namespaced keyword:

```clojure
(ig/annotate :adapter/jetty
  {:doc "A Ring adapter for the Jetty webserver."
   :schema {:port nat-int?}})
```

Retrieve annotations with `ig/describe`:

```clojure
(ig/describe :adapter/jetty)
;; => {:doc "A Ring adapter for the Jetty webserver." :schema {:port nat-int?}}
```

Annotations live in a global registry. They are optional but useful for tooling, documentation generation, and validation.

---

## load-namespaces

`load-namespaces` attempts to `require` namespaces derived from the keys in a configuration. For a key `:foo.component/bar`, it tries to load `foo.component` and `foo.component.bar`. Returns a list of successfully loaded namespaces (missing ones are silently ignored).

```clojure
(ig/load-namespaces config)
```

Call this before `ig/init` when your multimethods are spread across many namespaces, so you don't have to manually `require` each one.

**Naming convention to make this work:** if you define `init-key` for `:myapp.db/pool` in `myapp/db/pool.clj`, `load-namespaces` will find it automatically.

---

## load-hierarchy and load-annotations

Loading namespaces can be slow or side-effectful. For libraries that define keyword hierarchies and annotations, Integrant can load them from EDN files on the classpath instead:

```clojure
(ig/load-hierarchy)      ; searches classpath for integrant/hierarchy.edn
(ig/load-annotations)    ; searches classpath for integrant/annotations.edn
```

**`integrant/hierarchy.edn`** format:
```edn
{:adapter/jetty [:adapter/ring]}   ; :adapter/jetty derives from :adapter/ring
```

**`integrant/annotations.edn`** format:
```edn
{:adapter/jetty {:doc "Jetty Ring adapter"}}
```

Library authors should ship these files so downstream users can discover hierarchy and annotation information without loading (and executing) full namespaces.

---

## assert-key

Validate the initialized value of a key immediately after `init-key` returns. Use `assert` or throw explicitly.

```clojure
(defmethod ig/assert-key :adapter/jetty [_ {:keys [port]}]
  (assert (nat-int? port) ":port must be a non-negative integer"))

(defmethod ig/assert-key :db/pool [_ pool]
  (when-not (pool-healthy? pool)
    (throw (ex-info "DB pool failed health check" {:pool pool}))))
```

The error is wrapped in `ExceptionInfo`:

```clojure
;; ex-data keys:
{:reason :integrant.core/build-failed-spec
 :system  {...}   ; partial system built so far
 :key     :adapter/jetty
 :value   {:port "not-a-number"}}
```

---

## normalize-key

Returns the canonical form of a composite key. Useful for dispatching in your own tooling:

```clojure
(ig/normalize-key [:adapter/jetty :example/web-1])
;; => :integrant.composite/...  (a synthetic derived keyword)
```

For simple (non-composite) keys, returns the key unchanged. You rarely need this directly — it is used internally by Integrant's dispatch.
