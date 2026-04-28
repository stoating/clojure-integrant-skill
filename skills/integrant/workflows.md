# Integrant Workflows

## Contents
- Typical application setup
- Loading config from EDN
- REPL workflow with integrant-repl
- Testing an Integrant system
- Building a module with expand-key
- Migrating from Component

---

## Typical Application Setup

A minimal but complete Integrant application:

**`src/myapp/system.clj`** — define multimethods:
```clojure
(ns myapp.system
  (:require [integrant.core :as ig]
            [ring.adapter.jetty :as jetty]
            [ring.util.response :as resp]))

(defmethod ig/init-key :adapter/jetty [_ {:keys [handler port]}]
  (jetty/run-jetty handler {:port port :join? false}))

(defmethod ig/halt-key! :adapter/jetty [_ server]
  (.stop server))

(defmethod ig/init-key :handler/app [_ {:keys [name]}]
  (fn [_] (resp/response (str "Hello " name))))
```

**`resources/config.edn`** — configuration:
```edn
{:adapter/jetty {:port 8080, :handler #ig/ref :handler/app}
 :handler/app   {:name "World"}}
```

**`src/myapp/main.clj`** — entry point:
```clojure
(ns myapp.main
  (:require [integrant.core :as ig]
            [myapp.system]))   ; loads the defmethod implementations

(defn -main [& _]
  (let [config (-> (io/resource "config.edn") slurp ig/read-string)]
    (ig/load-namespaces config)   ; optional: auto-require ns from key names
    (ig/init config)))
```

---

## Loading Config from EDN

```clojure
(require '[clojure.java.io :as io]
         '[integrant.core :as ig])

;; From classpath resource
(def config (-> "config.edn" io/resource slurp ig/read-string))

;; From file
(def config (-> "config.edn" slurp ig/read-string))

;; From string (e.g. in tests)
(def config (ig/read-string "{:handler/greet {:name \"Alice\"}}"))
```

`ig/read-string` recognizes the `#ig/ref`, `#ig/refset`, `#ig/profile`, and `#ig/var` tagged literals. Standard `clojure.edn/read-string` does not.

---

## REPL Workflow with integrant-repl

[integrant-repl](https://github.com/weavejester/integrant-repl) provides a smooth workflow aligned with Stuart Sierra's reloaded pattern.

**deps.edn:**
```clojure
{:aliases
 {:dev {:extra-deps {integrant/integrant-repl {:mvn/version "0.3.3"}}}}}
```

**`dev/user.clj`:**
```clojure
(ns user
  (:require [integrant.repl :refer [clear go halt prep reset reset-all]]
            [integrant.repl.state :refer [config system]]
            [integrant.core :as ig]
            [myapp.system]))   ; loads defmethod impls

(integrant.repl/set-prep!
  (fn []
    (-> "dev/config.edn" slurp ig/read-string)))
```

**REPL commands:**

| Command | What it does |
|---------|-------------|
| `(go)` | Prep config and init the system |
| `(halt)` | Halt the running system |
| `(reset)` | `halt` → reload changed namespaces → `go` |
| `(reset-all)` | `halt` → reload **all** namespaces → `go` |
| `(prep)` | Load config without initing (sets `config`) |
| `(clear)` | Clear system and config state |

`system` and `config` in `integrant.repl.state` hold the live values.

---

## Testing an Integrant System

### Isolation pattern: init a subset

Test only the keys you need, avoiding expensive resources:

```clojure
(deftest handler-test
  (let [system (ig/init config [:handler/app])]
    (try
      (is (= 200 (:status ((get system :handler/app) {}))))
      (finally
        (ig/halt! system)))))
```

### Swap config values for test doubles

```clojure
(def test-config
  (assoc config :db/pool {:jdbcUrl "jdbc:h2:mem:test"}))

(deftest db-integration-test
  (let [system (ig/init test-config)]
    (try
      ;; tests...
      (finally (ig/halt! system)))))
```

### with-system helper (common pattern)

```clojure
(defmacro with-system [binding-form config & body]
  `(let [system# (ig/init ~config)
         ~binding-form system#]
     (try ~@body
          (finally (ig/halt! system#)))))

(deftest my-test
  (with-system sys test-config
    (is (some? (:adapter/jetty sys)))))
```

---

## Building a Module with expand-key

Use `expand-key` to bundle related keys into a reusable module:

```clojure
;; Define the module
(defmethod ig/expand-key :module/web [_ {:keys [port name]}]
  {:adapter/jetty {:port port, :handler (ig/ref :handler/app)}
   :handler/app   {:name name}})

;; Use it — the module key is consumed, not part of the initialized system
(def config
  {:module/web {:port 8080, :name "World"}})

;; Override a value from the module
(def config-override
  {:module/web    {:port 8080, :name "World"}
   :adapter/jetty {:port 3000}})   ; overrides port from expansion
```

```clojure
;; Always call expand before init when using modules
(-> config ig/expand ig/init)
```

**`expand` does not run recursively** — if an expanded result contains more expansion keys, call `expand` again or chain them explicitly.

---

## Migrating from Component

| Component concept | Integrant equivalent |
|-------------------|---------------------|
| `defrecord` + protocol | `defmethod ig/init-key` |
| `component/start` | `ig/init` |
| `component/stop` | `ig/halt!` |
| `(component/using ...)` | `ig/ref` / `ig/refset` in config |
| `component/system-map` | configuration map literal |
| `component/system-using` | wiring expressed as refs in config, not in code |

Key differences:
- In Component, systems are built programmatically; in Integrant, they are declared as data.
- In Component, only records/maps can have dependencies; in Integrant, **any** value can be a dependency.
- In Integrant, the configuration is a first-class value you can inspect, serialize, and transform before initializing.
- There is no global state requirement — `ig/init` returns the system; you hold the reference yourself.
