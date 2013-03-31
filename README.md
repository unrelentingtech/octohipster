Current [semantic](http://semver.org/) version:

```clojure
[octohipster "0.2.0"]
```

# octohipster [![Build Status](https://travis-ci.org/myfreeweb/octohipster.png?branch=master)](https://travis-ci.org/myfreeweb/octohipster)

A REST framework for Clojure that allows you to easily build high performance web APIs that:

- support hypermedia ([HAL+JSON](http://stateless.co/hal_specification.html), [Collection+JSON](http://amundsen.com/media-types/collection/) and Link/Link-Template HTTP headers; you can use hypermedia tools like [Frenetic](http://dlindahl.github.com/frenetic/) to build clients for your API)
- have [Swagger](https://github.com/wordnik/swagger-core/wiki) documentation (*not
  available for now due to epic refactoring*)
- use [JSON Schema](http://json-schema.org) for validation *and* documentation
- have pagination
- are 100% [Ring](https://github.com/ring-clojure/ring); you can add [rate limiting](https://github.com/myfreeweb/ring-ratelimit), [authentication](https://github.com/cemerick/friend), [metrics](http://metrics-clojure.readthedocs.org/en/latest/ring.html) and more with just middleware.

Octohipster is based on [Liberator](https://github.com/clojure-liberator/liberator).

## Usage

```clojure
(ns example
  (:use [octohipster core routes mixins pagination]
        org.httpkit.server)
  (:import org.bson.types.ObjectId)
  (:require [monger.core :as mg]
            [monger.query :as mq]
            [monger.collection :as mc]
            monger.json))

(mg/connect!)
(mg/set-db! (mg/get-db "octohipster-example"))

(def contact-schema
  {:id "Contact"
   :type "object"
   :properties {:name {:type "string"}
                :phone {:type "integer"}}
   :required [:name]})

(defn contacts-count []
  (mc/count "contacts"))
(defn contacts-all []
  (mq/with-collection "contacts"
    (mq/find {})
    (mq/skip *skip*)
    (mq/limit *limit*)))
(defn contacts-find-by-id [x]
  (mc/find-map-by-id "contacts" (ObjectId. x)))
(defn contacts-insert! [x]
  (mc/insert "contacts" (assoc x :_id (ObjectId.))))
(defn contacts-update! [x old]
  (mc/update "contacts" old x :multi false))
(defn contacts-delete! [x]
  (mc/remove "contacts" x))

(defresource contact-collection
  :mixins [collection-resource]
  :clinks {:item ::contact-item}
  :data-key :contacts
  :exists? (fn [ctx] {:contacts (contacts-all)})
  :post! (fn [ctx] (-> ctx :request :non-query-params contacts-insert!))
  :count (fn [req] (contacts-count)))

(defresource contact-item
  :mixins [item-resource]
  :url "/{_id}"
  :clinks {:collection ::contact-collection}
  :data-key :contact
  :exists? (fn [ctx]
             (if-let [doc (-> ctx :request :route-params :_id contacts-find-by-id)]
               {:contact doc}))
  :put! (fn [ctx]
          (-> ctx :request :non-query-params (contacts-update! (:contact ctx)))
          {:contact (-> ctx :request :route-params :_id contacts-find-by-id)})
  :delete! (fn [ctx]
             (-> ctx :contact contacts-delete!)
             {:contact nil}))

(defgroup contact-group
  :url "/contacts"
  :add-to-resources {:schema contact-schema}
  :resources [contact-collection contact-item])

(defroutes site
  :groups [contact-group])

(run-server site {:port 8080})
```

- [API Documentation](http://myfreeweb.github.com/octohipster) is available

## License

Copyright 2013 Greg V <floatboth@me.com>

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

[http://www.apache.org/licenses/LICENSE-2.0](http://www.apache.org/licenses/LICENSE-2.0)

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
