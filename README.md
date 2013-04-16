angular-defection
=================

### My sad experience with AngularJS

I had a need to build a ui based on some metadata. At first look Angular has all tools for that - looping, conditions, interpolated expressions. Let's try to do that.

Metadata.
```javascript
$ cat script.js
function Ctrl($scope) {
        $scope.params = [{
                id: "param1",
                type: "string"
        }, {
                id: "param2",
                type: "string"
        }];
};
```
Two string fields, simple. Now markup building ui.
```html
$ cat form.html
```
Styling the form with Bootstrap. Giving a name to the form allows us to use it later in expressions.
```html
<form class="form-horizontal" name="form">
```
This block will be repeated for each parameter in the `params` array. `ng-class` will mark the block with the `error` class, if the control for this parameter is invalid.
```html
        <div class="control-group" ng-repeat="param in params" ng-class="{ error : form.{{param.id}}ctrl.$invalid }">
```
Just a label for debugging purposes.
```html
                <label class="control-label">{{param.id}} {{param.value}}</label>
```
We can render different html pieces depending on the type of the parameter.
```html
                <div class="controls" ng-switch on="param.type">
                        <div ng-switch-when="string">
```
Assign a unique name to the control, so that we can access it in other places.
```html
                                <input name="{{param.id}}ctrl" type="text" ng-model="param.value" required>
```
Error message, visible only when corresponding control is invalid.
```html
                                <span class="help-inline" ng-show="form.{{param.id}}ctrl.$invalid">required</span>
                        </div>
```
Cases for other types are missing, one is enough for proof of concept.
```html
                </div>
        </div>
</form>
```

Looks great so far. But doesn't work. Cause `ngModelDirective` which registers controls in the form works before attribute interpolation directives, which update attributes after compilation in `$digest` loop.
```javascript
function addAttrInterpolateDirective(node, directives, value, name) {
    ...
    (attr.$$observers && attr.$$observers[name].$$scope || scope).$watch(interpolateFn, function interpolateFnWatchAction(value) {
        attr.$set(name, value);
    });
```
In fact all three used directives don't care about interpolation at all, I had to hack them to get working example (those are really hacks made without proper knowledge of internals, expect bugs and memory leaks mostly because of unregistered listeners).
```javascript
$ git diff pristine fix1-ngmodel
@@ -12134,7 +12134,13 @@ var ngModelDirective = function() {
       var modelCtrl = ctrls[0],
           formCtrl = ctrls[1] || nullFormCtrl;

-      formCtrl.$addControl(modelCtrl);
+      attr.$observe('name', function(value) {
+        if (modelCtrl.$name) {
+            formCtrl.$removeControl(modelCtrl);
+        }
+        modelCtrl.$name = value;
+        formCtrl.$addControl(modelCtrl);
+      });

$ git diff fix1-ngmodel fix2-ngshow
 var ngShowDirective = ngDirective(function(scope, element, attr){
-  scope.$watch(attr.ngShow, function ngShowWatchAction(value){
-    element.css('display', toBoolean(value) ? '' : 'none');
+  attr.$observe('ngShow', function(v) {
+    scope.$watch(v, function ngShowWatchAction(value){
+      element.css('display', toBoolean(value) ? '' : 'none');
+    });
   });
 });

$ git diff fix2-ngshow fix3-ngclass
@@ -12476,14 +12476,15 @@ function classDirective(name, selector) {
   return ngDirective(function(scope, element, attr) {
     var oldVal = undefined;

-    scope.$watch(attr[name], ngClassWatchAction, true);
+    attr.$observe('ngClass', function(v) {
+      scope.$watch(v, ngClassWatchAction, true);
+    });
```
Drat Angular! You are not easy to learn at all, I spent few days reading documentation, coding, searching, reading sources only to get to the conclusion, that what I want is impossible.
