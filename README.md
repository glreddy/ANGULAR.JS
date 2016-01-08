**
 + * @license AngularJS v1.3.7
 + * (c) 2010-2014 Google, Inc. http://angularjs.org
 + * License: MIT
 + */
 +
 +(function() {'use strict';
 +
 +/**
 + * @description
 + *
 + * This object provides a utility for producing rich Error messages within
 + * Angular. It can be called as follows:
 + *
 + * var exampleMinErr = minErr('example');
 + * throw exampleMinErr('one', 'This {0} is {1}', foo, bar);
 + *
 + * The above creates an instance of minErr in the example namespace. The
 + * resulting error will have a namespaced error code of example.one.  The
 + * resulting error will replace {0} with the value of foo, and {1} with the
 + * value of bar. The object is not restricted in the number of arguments it can
 + * take.
 + *
 + * If fewer arguments are specified than necessary for interpolation, the extra
 + * interpolation markers will be preserved in the final string.
 + *
 + * Since data will be parsed statically during a build step, some restrictions
 + * are applied with respect to how minErr instances are created and called.
 + * Instances should have names of the form namespaceMinErr for a minErr created
 + * using minErr('namespace') . Error codes, namespaces and template strings
 + * should all be static strings, not variables or general expressions.
 + *
 + * @param {string} module The namespace to use for the new minErr instance.
 + * @param {function} ErrorConstructor Custom error constructor to be instantiated when returning
 + *   error from returned function, for cases when a particular type of error is useful.
 + * @returns {function(code:string, template:string, ...templateArgs): Error} minErr instance
 + */
 +
 +function minErr(module, ErrorConstructor) {
 +  ErrorConstructor = ErrorConstructor || Error;
 +  return function() {
 +    var code = arguments[0],
 +      prefix = '[' + (module ? module + ':' : '') + code + '] ',
 +      template = arguments[1],
 +      templateArgs = arguments,
 +
 +      message, i;
 +
 +    message = prefix + template.replace(/\{\d+\}/g, function(match) {
 +      var index = +match.slice(1, -1), arg;
 +
 +      if (index + 2 < templateArgs.length) {
 +        return toDebugString(templateArgs[index + 2]);
 +      }
 +      return match;
 +    });
 +
 +    message = message + '\nhttp://errors.angularjs.org/1.3.7/' +
 +      (module ? module + '/' : '') + code;
 +    for (i = 2; i < arguments.length; i++) {
 +      message = message + (i == 2 ? '?' : '&') + 'p' + (i - 2) + '=' +
 +        encodeURIComponent(toDebugString(arguments[i]));
 +    }
 +    return new ErrorConstructor(message);
 +  };
 +}
 +
 +/**
 + * @ngdoc type
 + * @name angular.Module
 + * @module ng
 + * @description
 + *
 + * Interface for configuring angular {@link angular.module modules}.
 + */
 +
 +function setupModuleLoader(window) {
 +
 +  var $injectorMinErr = minErr('$injector');
 +  var ngMinErr = minErr('ng');
 +
 +  function ensure(obj, name, factory) {
 +    return obj[name] || (obj[name] = factory());
 +  }
 +
 +  var angular = ensure(window, 'angular', Object);
 +
 +  // We need to expose `angular.$$minErr` to modules such as `ngResource` that reference it during bootstrap
 +  angular.$$minErr = angular.$$minErr || minErr;
 +
 +  return ensure(angular, 'module', function() {
 +    /** @type {Object.<string, angular.Module>} */
 +    var modules = {};
 +
 +    /**
 +     * @ngdoc function
 +     * @name angular.module
 +     * @module ng
 +     * @description
 +     *
 +     * The `angular.module` is a global place for creating, registering and retrieving Angular
 +     * modules.
 +     * All modules (angular core or 3rd party) that should be available to an application must be
 +     * registered using this mechanism.
 +     *
 +     * When passed two or more arguments, a new module is created.  If passed only one argument, an
 +     * existing module (the name passed as the first argument to `module`) is retrieved.
 +     *
 +     *
 +     * # Module
 +     *
 +     * A module is a collection of services, directives, controllers, filters, and configuration information.
 +     * `angular.module` is used to configure the {@link auto.$injector $injector}.
 +     *
 +     * ```js
 +     * // Create a new module
 +     * var myModule = angular.module('myModule', []);
 +     *
 +     * // register a new service
 +     * myModule.value('appName', 'MyCoolApp');
 +     *
 +     * // configure existing services inside initialization blocks.
 +     * myModule.config(['$locationProvider', function($locationProvider) {
 +     *   // Configure existing providers
 +     *   $locationProvider.hashPrefix('!');
 +     * }]);
 +     * ```
 +     *
 +     * Then you can create an injector and load your modules like this:
 +     *
 +     * ```js
 +     * var injector = angular.injector(['ng', 'myModule'])
 +     * ```
 +     *
 +     * However it's more likely that you'll just use
 +     * {@link ng.directive:ngApp ngApp} or
 +     * {@link angular.bootstrap} to simplify this process for you.
 +     *
 +     * @param {!string} name The name of the module to create or retrieve.
 +     * @param {!Array.<string>=} requires If specified then new module is being created. If
 +     *        unspecified then the module is being retrieved for further configuration.
 +     * @param {Function=} configFn Optional configuration function for the module. Same as
 +     *        {@link angular.Module#config Module#config()}.
 +     * @returns {module} new module with the {@link angular.Module} api.
 +     */
 +    return function module(name, requires, configFn) {
 +      var assertNotHasOwnProperty = function(name, context) {
 +        if (name === 'hasOwnProperty') {
 +          throw ngMinErr('badname', 'hasOwnProperty is not a valid {0} name', context);
 +        }
 +      };
 +
 +      assertNotHasOwnProperty(name, 'module');
 +      if (requires && modules.hasOwnProperty(name)) {
 +        modules[name] = null;
 +      }
 +      return ensure(modules, name, function() {
 +        if (!requires) {
 +          throw $injectorMinErr('nomod', "Module '{0}' is not available! You either misspelled " +
 +             "the module name or forgot to load it. If registering a module ensure that you " +
 +             "specify the dependencies as the second argument.", name);
 +        }
 +
 +        /** @type {!Array.<Array.<*>>} */
 +        var invokeQueue = [];
 +
 +        /** @type {!Array.<Function>} */
 +        var configBlocks = [];
 +
 +        /** @type {!Array.<Function>} */
 +        var runBlocks = [];
 +
 +        var config = invokeLater('$injector', 'invoke', 'push', configBlocks);
 +
 +        /** @type {angular.Module} */
 +        var moduleInstance = {
 +          // Private state
 +          _invokeQueue: invokeQueue,
 +          _configBlocks: configBlocks,
 +          _runBlocks: runBlocks,
 +
 +          /**
 +           * @ngdoc property
 +           * @name angular.Module#requires
 +           * @module ng
 +           *
 +           * @description
 +           * Holds the list of modules which the injector will load before the current module is
 +           * loaded.
 +           */
 +          requires: requires,
 +
 +          /**
 +           * @ngdoc property
 +           * @name angular.Module#name
 +           * @module ng
 +           *
 +           * @description
 +           * Name of the module.
 +           */
 +          name: name,
 +
 +
 +          /**
 +           * @ngdoc method
 +           * @name angular.Module#provider
 +           * @module ng
 +           * @param {string} name service name
 +           * @param {Function} providerType Construction function for creating new instance of the
 +           *                                service.
 +           * @description
 +           * See {@link auto.$provide#provider $provide.provider()}.
 +           */
 +          provider: invokeLater('$provide', 'provider'),
 +
 +          /**
 +           * @ngdoc method
 +           * @name angular.Module#factory
 +           * @module ng
 +           * @param {string} name service name
 +           * @param {Function} providerFunction Function for creating new instance of the service.
 +           * @description
 +           * See {@link auto.$provide#factory $provide.factory()}.
 +           */
 +          factory: invokeLater('$provide', 'factory'),
 +
 +          /**
 +           * @ngdoc method
 +           * @name angular.Module#service
 +           * @module ng
 +           * @param {string} name service name
 +           * @param {Function} constructor A constructor function that will be instantiated.
 +           * @description
 +           * See {@link auto.$provide#service $provide.service()}.
 +           */
 +          service: invokeLater('$provide', 'service'),
 +
 +          /**
 +           * @ngdoc method
 +           * @name angular.Module#value
 +           * @module ng
 +           * @param {string} name service name
 +           * @param {*} object Service instance object.
 +           * @description
 +           * See {@link auto.$provide#value $provide.value()}.
 +           */
 +          value: invokeLater('$provide', 'value'),
 +
 +          /**
 +           * @ngdoc method
 +           * @name angular.Module#constant
 +           * @module ng
 +           * @param {string} name constant name
 +           * @param {*} object Constant value.
 +           * @description
 +           * Because the constant are fixed, they get applied before other provide methods.
 +           * See {@link auto.$provide#constant $provide.constant()}.
 +           */
 +          constant: invokeLater('$provide', 'constant', 'unshift'),
 +
 +          /**
 +           * @ngdoc method
 +           * @name angular.Module#animation
 +           * @module ng
 +           * @param {string} name animation name
 +           * @param {Function} animationFactory Factory function for creating new instance of an
 +           *                                    animation.
 +           * @description
 +           *
 +           * **NOTE**: animations take effect only if the **ngAnimate** module is loaded.
 +           *
 +           *
 +           * Defines an animation hook that can be later used with
 +           * {@link ngAnimate.$animate $animate} service and directives that use this service.
 +           *
 +           * ```js
 +           * module.animation('.animation-name', function($inject1, $inject2) {
 +           *   return {
 +           *     eventName : function(element, done) {
 +           *       //code to run the animation
 +           *       //once complete, then run done()
 +           *       return function cancellationFunction(element) {
 +           *         //code to cancel the animation
 +           *       }
 +           *     }
 +           *   }
 +           * })
 +           * ```
 +           *
 +           * See {@link ng.$animateProvider#register $animateProvider.register()} and
 +           * {@link ngAnimate ngAnimate module} for more information.
 +           */
 +          animation: invokeLater('$animateProvider', 'register'),
 +
 +          /**
 +           * @ngdoc method
 +           * @name angular.Module#filter
 +           * @module ng
 +           * @param {string} name Filter name.
 +           * @param {Function} filterFactory Factory function for creating new instance of filter.
 +           * @description
 +           * See {@link ng.$filterProvider#register $filterProvider.register()}.
 +           */
 +          filter: invokeLater('$filterProvider', 'register'),
 +
 +          /**
 +           * @ngdoc method
 +           * @name angular.Module#controller
 +           * @module ng
 +           * @param {string|Object} name Controller name, or an object map of controllers where the
 +           *    keys are the names and the values are the constructors.
 +           * @param {Function} constructor Controller constructor function.
 +           * @description
 +           * See {@link ng.$controllerProvider#register $controllerProvider.register()}.
 +           */
 +          controller: invokeLater('$controllerProvider', 'register'),
 +
 +          /**
 +           * @ngdoc method
 +           * @name angular.Module#directive
 +           * @module ng
 +           * @param {string|Object} name Directive name, or an object map of directives where the
 +           *    keys are the names and the values are the factories.
 +           * @param {Function} directiveFactory Factory function for creating new instance of
 +           * directives.
 +           * @description
 +           * See {@link ng.$compileProvider#directive $compileProvider.directive()}.
 +           */
 +          directive: invokeLater('$compileProvider', 'directive'),
 +
 +          /**
 +           * @ngdoc method
 +           * @name angular.Module#config
 +           * @module ng
 +           * @param {Function} configFn Execute this function on module load. Useful for service
 +           *    configuration.
 +           * @description
 +           * Use this method to register work which needs to be performed on module loading.
 +           * For more about how to configure services, see
 +           * {@link providers#provider-recipe Provider Recipe}.
 +           */
 +          config: config,
 +
 +          /**
 +           * @ngdoc method
 +           * @name angular.Module#run
 +           * @module ng
 +           * @param {Function} initializationFn Execute this function after injector creation.
 +           *    Useful for application initialization.
 +           * @description
 +           * Use this method to register work which should be performed when the injector is done
 +           * loading all modules.
 +           */
 +          run: function(block) {
 +            runBlocks.push(block);
 +            return this;
 +          }
 +        };
 +
 +        if (configFn) {
 +          config(configFn);
 +        }
 +
 +        return moduleInstance;
 +
 +        /**
 +         * @param {string} provider
 +         * @param {string} method
 +         * @param {String=} insertMethod
 +         * @returns {angular.Module}
 +         */
 +        function invokeLater(provider, method, insertMethod, queue) {
 +          if (!queue) queue = invokeQueue;
 +          return function() {
 +            queue[insertMethod || 'push']([provider, method, arguments]);
 +            return moduleInstance;
 +          };
 +        }
 +      });
 +    };
 +  });
 +
 +}
 +
 +setupModuleLoader(window);
 +})(window);
 +
 +/**
 + * Closure compiler type information
 + *
 + * @typedef { {
 + *   requires: !Array.<string>,
 + *   invokeQueue: !Array.<Array.<*>>,
 + *
 + *   service: function(string, Function):angular.Module,
 + *   factory: function(string, Function):angular.Module,
 + *   value: function(string, *):angular.Module,
 + *
 + *   filter: function(string, Function):angular.Module,
 + *
 + *   init: function(Function):angular.Module
 + * } }
 + */
 +angular.Module;
 +
  @license AngularJS v1.3.7
 + * (c) 2010-2014 Google, Inc. http://angularjs.org
 + * License: MIT
 + */
 +(function(window, angular, undefined) {'use strict';
 +
 +/**
 + * @ngdoc module
 + * @name ngAria
 + * @description
 + *
 + * The `ngAria` module provides support for common
 + * [<abbr title="Accessible Rich Internet Applications">ARIA</abbr>](http://www.w3.org/TR/wai-aria/)
 + * attributes that convey state or semantic information about the application for users
 + * of assistive technologies, such as screen readers.
 + *
 + * <div doc-module-components="ngAria"></div>
 + *
 + * ## Usage
 + *
 + * For ngAria to do its magic, simply include the module as a dependency. The directives supported
 + * by ngAria are:
 + * `ngModel`, `ngDisabled`, `ngShow`, `ngHide`, `ngClick`, `ngDblClick`, and `ngMessages`.
 + *
 + * Below is a more detailed breakdown of the attributes handled by ngAria:
 + *
 + * | Directive                                   | Supported Attributes                                                                   |
 + * |---------------------------------------------|----------------------------------------------------------------------------------------|
 + * | {@link ng.directive:ngModel ngModel}        | aria-checked, aria-valuemin, aria-valuemax, aria-valuenow, aria-invalid, aria-required |
 + * | {@link ng.directive:ngDisabled ngDisabled}  | aria-disabled                                                                          |
 + * | {@link ng.directive:ngShow ngShow}          | aria-hidden                                                                            |
 + * | {@link ng.directive:ngHide ngHide}          | aria-hidden                                                                            |
 + * | {@link ng.directive:ngClick ngClick}        | tabindex, keypress event                                                               |
 + * | {@link ng.directive:ngDblclick ngDblclick}  | tabindex                                                                               |
 + * | {@link module:ngMessages ngMessages}        | aria-live                                                                              |
 + *
 + * Find out more information about each directive by reading the
 + * {@link guide/accessibility ngAria Developer Guide}.
 + *
 + * ##Example
 + * Using ngDisabled with ngAria:
 + * ```html
 + * <md-checkbox ng-disabled="disabled">
 + * ```
 + * Becomes:
 + * ```html
 + * <md-checkbox ng-disabled="disabled" aria-disabled="true">
 + * ```
 + *
 + * ##Disabling Attributes
 + * It's possible to disable individual attributes added by ngAria with the
 + * {@link ngAria.$ariaProvider#config config} method. For more details, see the
 + * {@link guide/accessibility Developer Guide}.
 + */
 + /* global -ngAriaModule */
 +var ngAriaModule = angular.module('ngAria', ['ng']).
 +                        provider('$aria', $AriaProvider);
 +
 +/**
 + * @ngdoc provider
 + * @name $ariaProvider
 + *
 + * @description
 + *
 + * Used for configuring the ARIA attributes injected and managed by ngAria.
 + *
 + * ```js
 + * angular.module('myApp', ['ngAria'], function config($ariaProvider) {
 + *   $ariaProvider.config({
 + *     ariaValue: true,
 + *     tabindex: false
 + *   });
 + * });
 + *```
 + *
 + * ## Dependencies
 + * Requires the {@link ngAria} module to be installed.
 + *
 + */
 +function $AriaProvider() {
 +  var config = {
 +    ariaHidden: true,
 +    ariaChecked: true,
 +    ariaDisabled: true,
 +    ariaRequired: true,
 +    ariaInvalid: true,
 +    ariaMultiline: true,
 +    ariaValue: true,
 +    tabindex: true,
 +    bindKeypress: true
 +  };
 +
 +  /**
 +   * @ngdoc method
 +   * @name $ariaProvider#config
 +   *
 +   * @param {object} config object to enable/disable specific ARIA attributes
 +   *
 +   *  - **ariaHidden** – `{boolean}` – Enables/disables aria-hidden tags
 +   *  - **ariaChecked** – `{boolean}` – Enables/disables aria-checked tags
 +   *  - **ariaDisabled** – `{boolean}` – Enables/disables aria-disabled tags
 +   *  - **ariaRequired** – `{boolean}` – Enables/disables aria-required tags
 +   *  - **ariaInvalid** – `{boolean}` – Enables/disables aria-invalid tags
 +   *  - **ariaMultiline** – `{boolean}` – Enables/disables aria-multiline tags
 +   *  - **ariaValue** – `{boolean}` – Enables/disables aria-valuemin, aria-valuemax and aria-valuenow tags
 +   *  - **tabindex** – `{boolean}` – Enables/disables tabindex tags
 +   *  - **bindKeypress** – `{boolean}` – Enables/disables keypress event binding on ng-click
 +   *
 +   * @description
 +   * Enables/disables various ARIA attributes
 +   */
 +  this.config = function(newConfig) {
 +    config = angular.extend(config, newConfig);
 +  };
 +
 +  function watchExpr(attrName, ariaAttr, negate) {
 +    return function(scope, elem, attr) {
 +      var ariaCamelName = attr.$normalize(ariaAttr);
 +      if (config[ariaCamelName] && !attr[ariaCamelName]) {
 +        scope.$watch(attr[attrName], function(boolVal) {
 +          if (negate) {
 +            boolVal = !boolVal;
 +          }
 +          elem.attr(ariaAttr, boolVal);
 +        });
 +      }
 +    };
 +  }
 +
 +  /**
 +   * @ngdoc service
 +   * @name $aria
 +   *
 +   * @description
 +   *
 +   * The $aria service contains helper methods for applying common
 +   * [ARIA](http://www.w3.org/TR/wai-aria/) attributes to HTML directives.
 +   *
 +   * ngAria injects common accessibility attributes that tell assistive technologies when HTML
 +   * elements are enabled, selected, hidden, and more. To see how this is performed with ngAria,
 +   * let's review a code snippet from ngAria itself:
 +   *
 +   *```js
 +   * ngAriaModule.directive('ngDisabled', ['$aria', function($aria) {
 +   *   return $aria.$$watchExpr('ngDisabled', 'aria-disabled');
 +   * }])
 +   *```
 +   * Shown above, the ngAria module creates a directive with the same signature as the
 +   * traditional `ng-disabled` directive. But this ngAria version is dedicated to
 +   * solely managing accessibility attributes. The internal `$aria` service is used to watch the
 +   * boolean attribute `ngDisabled`. If it has not been explicitly set by the developer,
 +   * `aria-disabled` is injected as an attribute with its value synchronized to the value in
 +   * `ngDisabled`.
 +   *
 +   * Because ngAria hooks into the `ng-disabled` directive, developers do not have to do
 +   * anything to enable this feature. The `aria-disabled` attribute is automatically managed
 +   * simply as a silent side-effect of using `ng-disabled` with the ngAria module.
 +   *
 +   * The full list of directives that interface with ngAria:
 +   * * **ngModel**
 +   * * **ngShow**
 +   * * **ngHide**
 +   * * **ngClick**
 +   * * **ngDblclick**
 +   * * **ngMessages**
 +   * * **ngDisabled**
 +   *
 +   * Read the {@link guide/accessibility ngAria Developer Guide} for a thorough explanation of each
 +   * directive.
 +   *
 +   *
 +   * ## Dependencies
 +   * Requires the {@link ngAria} module to be installed.
 +   */
 +  this.$get = function() {
 +    return {
 +      config: function(key) {
 +        return config[key];
 +      },
 +      $$watchExpr: watchExpr
 +    };
 +  };
 +}
 +
 +
 +ngAriaModule.directive('ngShow', ['$aria', function($aria) {
 +  return $aria.$$watchExpr('ngShow', 'aria-hidden', true);
 +}])
 +.directive('ngHide', ['$aria', function($aria) {
 +  return $aria.$$watchExpr('ngHide', 'aria-hidden', false);
 +}])
 +.directive('ngModel', ['$aria', function($aria) {
 +
 +  function shouldAttachAttr(attr, normalizedAttr, elem) {
 +    return $aria.config(normalizedAttr) && !elem.attr(attr);
 +  }
 +
 +  function getShape(attr, elem) {
 +    var type = attr.type,
 +        role = attr.role;
 +
 +    return ((type || role) === 'checkbox' || role === 'menuitemcheckbox') ? 'checkbox' :
 +           ((type || role) === 'radio'    || role === 'menuitemradio') ? 'radio' :
 +           (type === 'range'              || role === 'progressbar' || role === 'slider') ? 'range' :
 +           (type || role) === 'textbox'   || elem[0].nodeName === 'TEXTAREA' ? 'multiline' : '';
 +  }
 +
 +  return {
 +    restrict: 'A',
 +    require: '?ngModel',
 +    link: function(scope, elem, attr, ngModel) {
 +      var shape = getShape(attr, elem);
 +      var needsTabIndex = shouldAttachAttr('tabindex', 'tabindex', elem);
 +
 +      function ngAriaWatchModelValue() {
 +        return ngModel.$modelValue;
 +      }
 +
 +      function getRadioReaction() {
 +        if (needsTabIndex) {
 +          needsTabIndex = false;
 +          return function ngAriaRadioReaction(newVal) {
 +            var boolVal = newVal === attr.value;
 +            elem.attr('aria-checked', boolVal);
 +            elem.attr('tabindex', 0 - !boolVal);
 +          };
 +        } else {
 +          return function ngAriaRadioReaction(newVal) {
 +            elem.attr('aria-checked', newVal === attr.value);
 +          };
 +        }
 +      }
 +
 +      function ngAriaCheckboxReaction(newVal) {
 +        elem.attr('aria-checked', !!newVal);
 +      }
 +
 +      switch (shape) {
 +        case 'radio':
 +        case 'checkbox':
 +          if (shouldAttachAttr('aria-checked', 'ariaChecked', elem)) {
 +            scope.$watch(ngAriaWatchModelValue, shape === 'radio' ?
 +                getRadioReaction() : ngAriaCheckboxReaction);
 +          }
 +          break;
 +        case 'range':
 +          if ($aria.config('ariaValue')) {
 +            if (attr.min && !elem.attr('aria-valuemin')) {
 +              elem.attr('aria-valuemin', attr.min);
 +            }
 +            if (attr.max && !elem.attr('aria-valuemax')) {
 +              elem.attr('aria-valuemax', attr.max);
 +            }
 +            if (!elem.attr('aria-valuenow')) {
 +              scope.$watch(ngAriaWatchModelValue, function ngAriaValueNowReaction(newVal) {
 +                elem.attr('aria-valuenow', newVal);
 +              });
 +            }
 +          }
 +          break;
 +        case 'multiline':
 +          if (shouldAttachAttr('aria-multiline', 'ariaMultiline', elem)) {
 +            elem.attr('aria-multiline', true);
 +          }
 +          break;
 +      }
 +
 +      if (needsTabIndex) {
 +        elem.attr('tabindex', 0);
 +      }
 +
 +      if (ngModel.$validators.required && shouldAttachAttr('aria-required', 'ariaRequired', elem)) {
 +        scope.$watch(function ngAriaRequiredWatch() {
 +          return ngModel.$error.required;
 +        }, function ngAriaRequiredReaction(newVal) {
 +          elem.attr('aria-required', !!newVal);
 +        });
 +      }
 +
 +      if (shouldAttachAttr('aria-invalid', 'ariaInvalid', elem)) {
 +        scope.$watch(function ngAriaInvalidWatch() {
 +          return ngModel.$invalid;
 +        }, function ngAriaInvalidReaction(newVal) {
 +          elem.attr('aria-invalid', !!newVal);
 +        });
 +      }
 +    }
 +  };
 +}])
 +.directive('ngDisabled', ['$aria', function($aria) {
 +  return $aria.$$watchExpr('ngDisabled', 'aria-disabled');
 +}])
 +.directive('ngMessages', function() {
 +  return {
 +    restrict: 'A',
 +    require: '?ngMessages',
 +    link: function(scope, elem, attr, ngMessages) {
 +      if (!elem.attr('aria-live')) {
 +        elem.attr('aria-live', 'assertive');
 +      }
 +    }
 +  };
 +})
 +.directive('ngClick',['$aria', function($aria) {
 +  return {
 +    restrict: 'A',
 +    link: function(scope, elem, attr) {
 +      if ($aria.config('tabindex') && !elem.attr('tabindex')) {
 +        elem.attr('tabindex', 0);
 +      }
 +
 +      if ($aria.config('bindKeypress') && !elem.attr('ng-keypress')) {
 +        elem.on('keypress', function(event) {
 +          if (event.keyCode === 32 || event.keyCode === 13) {
 +            scope.$eval(attr.ngClick);
 +          }
 +        });
 +      }
 +    }
 +  };
 +}])
 +.directive('ngDblclick', ['$aria', function($aria) {
 +  return function(scope, elem, attr) {
 +    if ($aria.config('tabindex') && !elem.attr('tabindex')) {
 +      elem.attr('tabindex', 0);
 +    }
 +  };
 +}]);
 +
 +
 +})(window, window.angular);
 ar http = require('http'),
 +    url = require('url'),
 +    join = require('path').join,
 +    exists = require('path').exists,
 +    extname = require('path').extname,
 +    join = require('path').join,
 +    fs = require('fs'),
 +    port = process.argv[2] || 3000
 +
 +var mime = {
 +    'html': 'text/html',
 +    'css': 'text/css',
 +    'js': 'application/javascript',
 +}
 +
 +http.createServer(function(req, res){
 +  console.log('  \033[90m%s \033[36m%s\033[m', req.method, req.url)
 +
 +  var pathname = url.parse(req.url).pathname,
 +      path = join(process.cwd(), pathname)
 +
 +  function notFound() {
 +    res.statusCode = 404
 +    res.end("404 Not Found\n")
 +  }
 +
 +  function error(err) {
 +    res.statusCode = 500
 +    res.end(err.message + "\n")
 +  }
 +
 +  exists(path, function(exists){
 +    if (!exists) return notFound()
 +
 +    fs.stat(path, function(err, stat){
 +      if (err) return error()
 +      if (stat.isDirectory()) path = join(path, 'index.html')
 +      res.setHeader('Cache-Control', 'no-cache')
 +      res.setHeader('Content-Type', mime[path.split('.').slice(-1)] || 'application/octet-stream')
 +      fs.createReadStream(path).pipe(res)
 +    })
 +  })
 +}).listen(port)
 +
 +console.log('\n  Server listening on %d\n', port)
 Pretty printing styles. Used with prettify.js. */
 +/* Vim sunburst theme by David Leibovic */
 +
 +pre .str, code .str { color: #65B042; } /* string  - green */
 +pre .kwd, code .kwd { color: #E28964; } /* keyword - dark pink */
 +pre .com, code .com { color: #AEAEAE; font-style: italic; } /* comment - gray */
 +pre .typ, code .typ { color: #89bdff; } /* type - light blue */
 +pre .lit, code .lit { color: #3387CC; } /* literal - blue */
 +pre .pun, code .pun { color: #fff; } /* punctuation - white */
 +pre .pln, code .pln { color: #fff; } /* plaintext - white */
 +pre .tag, code .tag { color: #89bdff; } /* html/xml tag    - light blue */
 +pre .atn, code .atn { color: #bdb76b; } /* html/xml attribute name  - khaki */
 +pre .atv, code .atv { color: #65B042; } /* html/xml attribute value - green */
 +pre .dec, code .dec { color: #3387CC; } /* decimal - blue */
 +
 +pre.prettyprint, code.prettyprint {
 +	background-color: #000;
 +	-moz-border-radius: 8px;
 +	-webkit-border-radius: 8px;
 +	-o-border-radius: 8px;
 +	-ms-border-radius: 8px;
 +	-khtml-border-radius: 8px;
 +	border-radius: 8px;
 +}
 +
 +pre.prettyprint {
 +	width: 95%;
 +	margin: 1em auto;
 +	padding: 1em;
 +	white-space: pre-wrap;
 +}
 +
 +
 +/* Specify class=linenums on a pre to get line numbering */
 +ol.linenums { margin-top: 0; margin-bottom: 0; color: #AEAEAE; } /* IE indents via margin-left */
 +li.L0,li.L1,li.L2,li.L3,li.L5,li.L6,li.L7,li.L8 { list-style-type: none }
 +/* Alternate shading for lines */
 +li.L1,li.L3,li.L5,li.L7,li.L9 { }
 +
 +@media print {
 +  pre .str, code .str { color: #060; }
 +  pre .kwd, code .kwd { color: #006; font-weight: bold; }
 +  pre .com, code .com { color: #600; font-style: italic; }
 +  pre .typ, code .typ { color: #404; font-weight: bold; }
 +  pre .lit, code .lit { color: #044; }
 +  pre .pun, code .pun { color: #440; }
 +  pre .pln, code .pln { color: #000; }
 +  pre .tag, code .tag { color: #006; font-weight: bold; }
 +  pre .atn, code .atn { color: #404; }
 +  pre .atv, code .atv { color: #060; }
 +}
