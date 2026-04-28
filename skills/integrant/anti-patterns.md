# Integrant Anti-Patterns

## Contents
- Configuration anti-patterns
- Lifecycle anti-patterns
- Dependency anti-patterns
- Multimethod anti-patterns
- Testing anti-patterns
- expand anti-patterns
- Quick diagnostic checklist

---

## Configuration Anti-Patterns

### Using unqualified keywords as keys

```clojure
;; WRONG — Integrant requires qualified (namespaced) keywords
(def config {:jetty {:port 8080}})

;; RIGHT
(def config {:adapter/jetty {:port 8080}})
```

Integrant throws an exception on `init` if any top-level key is unqualified.

---

### Reading EDN config with `clojure.edn/read-string`

```clojure
;; WRONG — standard read-string doesn't know #ig/ref, #ig/refset, #ig/profile, #ig/var
(def config (clojure.edn/read-string (slurp "config.edn")))

;; RIGHT — use Integrant's reader
(def config (ig/read-string (slurp "config.edn")))
```

The tagged literals are silently discarded or throw with the standard reader.

---

### Holding references to config values after init

```clojure
;; WRONG — (:db config) is the raw config map, not the initialized connection pool
(defn query []
  (jdbc/execute! (:db config) ["SELECT 1"]))

;; RIGHT — use the initialized system map
(defn query [system]
  (jdbc/execute! (:db/pool system) ["SELECT 1"]))
```

After `ig/init`, the system holds the initialized values; the config map remains unchanged.

---

## Lifecycle Anti-Patterns

### Ignoring the return value of `halt!` / `suspend!`

```clojure
;; WRONG — halt! returns the system but its return value should be discarded
(def system (ig/halt! system))   ; confusing and incorrect

;; RIGHT — discard and clear the reference
(ig/halt! system)
(def system nil)
```

---

### Not calling `halt!` on error during startup

If `init-key` throws for one key, Integrant already halted the partially-built system internally. But you should still not hold onto the failed return value:

```clojure
;; RIGHT — wrap init in try/catch; system is cleaned up internally on error
(def system
  (try (ig/init config)
       (catch Exception e
         (log/error e "System failed to start")
         nil)))
```

---

### Making `halt-key!` non-idempotent

```clojure
;; WRONG — throws if called twice (e.g. server already stopped)
(defmethod ig/halt-key! :adapter/jetty [_ server]
  (.stop server))   ; throws org.eclipse.jetty.server.Server: STOPPED if called again

;; RIGHT — check state before acting
(defmethod ig/halt-key! :adapter/jetty [_ server]
  (when (.isRunning server)
    (.stop server)))
```

`halt-key!` must be safe to call multiple times on the same value.

---

### Using `suspend!` / `resume` in production

`suspend!` and `resume` are development tools. In production, always use `halt!` and `init` for clean restarts. Keeping suspended resources alive in production leads to connection leaks and inconsistent state.

---

## Dependency Anti-Patterns

### Circular references

```clojure
;; WRONG — :service/a needs :service/b and :service/b needs :service/a
{:service/a {:dep (ig/ref :service/b)}
 :service/b {:dep (ig/ref :service/a)}}
```

Integrant throws a `weavejester.dependency.CircularDependencyException`. Break the cycle by extracting shared state into a third key (e.g. a shared `:state/cache`).

---

### Ambiguous refs

```clojure
;; WRONG — two keys derive from :adapter/ring; which one does the ref resolve to?
(derive :adapter/jetty :adapter/ring)
(derive :adapter/http-kit :adapter/ring)

{:handler/app {}
 :adapter/jetty    {:handler (ig/ref :handler/app)}
 :adapter/http-kit {:handler (ig/ref :handler/app)}
 :stats/collector  {:adapter (ig/ref :adapter/ring)}}   ; ambiguous — throws
```

Use a more specific key in the ref, or a composite key + composite ref to disambiguate.

---

### Using `ig/refset` when you expect exactly one value

```clojure
;; WRONG — if there is only one :db/conn, refset gives #{conn} not conn
{:service/query {:db #ig/refset :db/conn}}

(defmethod ig/init-key :service/query [_ {:keys [db]}]
  ;; db is a set, not the conn — this will break
  (jdbc/execute! db ["SELECT 1"]))

;; RIGHT — use ig/ref when you expect exactly one dependency
{:service/query {:db #ig/ref :db/conn}}
```

Use `ig/refset` only when you genuinely want a set of all matching initialized values.

---

## Multimethod Anti-Patterns

### Defining `init-key` in a namespace that isn't loaded

```clojure
;; myapp/db.clj defines:
(defmethod ig/init-key :myapp/db [_ opts] ...)

;; main.clj forgets to require it:
(ns myapp.main
  (:require [integrant.core :as ig]))
  ; missing: [myapp.db]

(ig/init {:myapp/db {...}})  ; falls through to default init-key (no-op or function lookup fails)
```

Either `require` the namespace explicitly or call `(ig/load-namespaces config)` before `ig/init`.

---

### Putting side effects in `expand-key`

```clojure
;; WRONG — expand-key is for transforming configuration, not starting resources
(defmethod ig/expand-key :module/db [_ opts]
  (let [conn (jdbc/get-connection opts)]    ; starts DB connection at expand time!
    {:db/conn {:connection conn}}))

;; RIGHT — expand-key returns config data; init-key starts resources
(defmethod ig/expand-key :module/db [_ opts]
  {:db/conn opts})

(defmethod ig/init-key :db/conn [_ opts]
  (jdbc/get-connection opts))
```

---

### Forgetting to call `expand` before `init` when using `expand-key`

```clojure
;; WRONG — module keys are not expanded; they are initialized as-is
(ig/init {:module/web {:port 8080}})

;; RIGHT
(-> {:module/web {:port 8080}} ig/expand ig/init)
```

Module keys that survive un-expanded into `init` will typically fail because no `init-key` method is defined for them.

---

## Testing Anti-Patterns

### Initializing the full system for unit tests

```clojure
;; WRONG — starts everything including DB, network, etc. for a pure handler test
(def system (ig/init full-production-config))

;; RIGHT — init only the keys under test
(def system (ig/init config [:handler/app]))
```

Pass a collection of keys to `ig/init` to limit initialization to only what you need.

---

### Not halting the system after tests

```clojure
;; WRONG — leaks server threads and connections across tests
(deftest my-test
  (let [system (ig/init config)]
    (is ...)))

;; RIGHT — always halt in finally
(deftest my-test
  (let [system (ig/init config)]
    (try
      (is ...)
      (finally (ig/halt! system)))))
```

---

## expand Anti-Patterns

### Conflicting expansions without an override

```clojure
;; :module/greet expands to {:adapter/jetty {:port 8080}}
;; :module/web-server expands to {:adapter/jetty {:port 80}}
;; Both try to set :adapter/jetty — Integrant throws

;; WRONG
(ig/expand {:module/greet {} :module/web-server {}})

;; RIGHT — provide an explicit override in the config
(ig/expand {:module/greet     {}
            :module/web-server {}
            :adapter/jetty    {:port 80}})   ; explicit value wins
```

---

## Quick Diagnostic Checklist

When something doesn't work:

```text
[ ] Are all top-level config keys qualified keywords?
[ ] Did you use ig/read-string (not clojure.edn/read-string) to load EDN?
[ ] Are all defmethod namespaces loaded before ig/init?
[ ] Did you call (ig/load-namespaces config) or require each namespace explicitly?
[ ] Are refs unambiguous? (only one key matches each #ig/ref target)
[ ] Did you call ig/expand before ig/init when using expand-key?
[ ] Is halt-key! idempotent?
[ ] Are you working with the system map (not the config map) after init?
[ ] Are there circular dependencies? (look for CircularDependencyException)
[ ] In tests: are you halting the system in a finally block?
[ ] Are unbound vars present? (ig/init throws if any #ig/var is unbound)
[ ] Did you call (ig/deprofile config [...]) before init if using #ig/profile?
```
