## TL;DR
An app that reminds you for holidays

### Article Link
https://notamonadtutorial.com/holiday-ping-how-we-implemented-our-first-open-source-app-with-erlang-and-clojurescript-fad5b66fc325
https://notamonadtutorial.com/one-does-not-simply-build-a-user-interface-our-clojurescript-re-frame-app-67b1354a2d55
### Author
notamonadtutorial
## Key Takeaways
* How to create a clojurescript app
* How to query, routing, navigation, forms, validations, and databases

## Useful Code Snippets
Navigation
```clojure
(def panels {:panel1 [panel1]
             :panel2 [panel2]})
(defn high-level-view 
  []
  (let [active (re-frame/subscribe [:active-panel])]
    (fn []
      [:div
       [:div.title   "Heading"]
       (get panels @active)])))

```
Events
```clojure
  (re-frame/reg-event-fx
 :navigate
 (fn [{:keys [db]} [_ new-view]]
   {;; trigger the load-view event in case data from the server is required
    :dispatch    [:load-view new-view] 
    ;; update the browser history API and switch to the given view
    :set-history new-view
    ;; set the current view in the app-db so the dom is updated
    :db          (assoc db
                        :loading-view? true
                        :current-view new-view)}))
(defmulti load-view
  "Create a multimethod that will implement different event handlers based on the
   view keyword."
  (fn [cofx [view]] view))
(re-frame/reg-event-fx
 :load-view
 (fn [cofx [_ new-view]]
   ;; delegate event handling to the proper multimethod
   (load-view cofx [vector new-view])))
(defmethod load-view 
  :default
  [{:keys [db]} _]
  ;; by default don't load anything
  {:db (assoc db :loading-view? false)})
(defmethod load-view
  :channel-list
  [_ _]
  ;; when navigating to the channel list, fetch the channels
  {:http-xhrio {:method     :get
                :uri        "/api/channels"
                :on-success [:channel-list-success]}})
```
Forms
``clojure
[forms/form-view {:submit-text "Register"
                  :on-submit   [:register-submit]
                  :fields      [{:key      :email
                                 :type     "email"
                                 :validate :valid-email?
                                 :required true}
                                {:key      :password
                                 :type     "password"
                                 :required true}
                                {:key      :password-repeat
                                 :type     "password"
                                 :label    "Repeat password"
                                 :validate :matching-passwords?
                                 :required true}]}]
```
Validations
```clojurescript
(re-frame/reg-sub
 :valid-required?
 (fn [db [_ value]]
   (if (string/blank? value)
     [false "This field is required."]
     [true])))
(re-frame/reg-sub
 :valid-form?
 (fn [[_ form fields]]
   (->> fields
        (get-validation-subs form)
        (map re-frame/subscribe)
        (doall)))
(fn [validations _]
   (every? true? (map first validations))))
```

## Useful Tools
* CRUD
* bidi and pushy for http
* Bulma for css

## Comments/ Questions
