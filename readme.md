Testing countries & capitals
=====================================

`karma init` installs jasmine 1.3 by default, 2.0 has cleaner syntax for asynchronous tests, but doesn't support all of the same libraries.

Testing an Asynchronous Service with Jasmine 2.0
-------------------------

If there is a service that uses promises, like:

    angular.module('myFactory', [])
    .constant('API_AUTH', '&username=demo')
    .constant('COUNTRIES_PATH', 'http://api.geonames.org/countryInfoJSON?')
    .factory('CountryData', [ 'API_AUTH','$http','$q', 'COUNTRIES_PATH',
    function (API_AUTH, $http, $q, COUNTRIES_PATH) {
        return function (countryId) {
            var defer = $q.defer();
            $http.get(COUNTRIES_PATH + '&country=' + countryId + API_AUTH, { cache: true }).success(function (data) {
                defer.resolve(data.geonames[0]);
            });
            return defer.promise;
        };
    }]);
    
Any tests for this service will not resolve automatically because of the promise:

    describe('myFactory', function(){
        it('never finishes', function(){
            inject(function(myService, $httpBackend){
      
                var responseData = 'hi'
                $httpBackend.expectGET(/http:\/\/\.api\.geonames\.com/).respond(responseData);
      
                myFactory().then(function(data){
                    expect(data).toBe('I can write anything here because myFactory will never resolve');
                });
            });
        });
    });
  
This test will never fail because `myFactory` never resolves. In jasmine 2.0 pass `done` into the `it` block and call it
to resolve the asyn call:

    it('finishes', function(done){
        inject(function(myFactory, $httpBackend){
      
            var responseData = 'hi'
            $httpBackend.expectGET(/http:\/\/myServiceAPIcall\.com/).respond(responseData);
      
            myFactory().then(function(data){
                expect(data).toBe('it fails');
                done();
            });
        });
    });

This test actully runs and fails because `myFactory` runs and returns `hi!`.

`Done` is only in jasmine 2.0!!!!!

http://jasmine.github.io/2.0/introduction.html#section-Asynchronous_Support
1.3 has a different and less intuitive syntax:
http://jasmine.github.io/1.3/introduction.html#section-Asynchronous_Support

####Loose-coupling on urls

Using the full URL path with `$httpBackend` couples the test to the code. 

    $httpBackend.expectGET('http://api.geonames.com/someendpoint?&username=demo&country=AU').respond(responseData);
    
This approach works, but if the URL ever needs to change in the service, then the test must also change.
Instead, use a minimal regex:

    $httpBackend.expectGET(/http:\/\/geonames\.com\.com/).respond(responseData);

As long as the URL the service calls matches that regex, `$httpBackend` will respond appropriately. Now, if the api endpoint or URL params change, the test doesn't need to change.

Routing tests
------------------

Given a basic route like:

    angular.module('cc-app')
    .config(['$routeProvider', function ($routeProvider) {
        $routeProvider.when('/countries/:id', {
            templateUrl: './country/country.html',
            controller: 'countryCtrl',
            resolve: {
                ActiveCountry: ['CountryData', '$route', function(CountryData, $route) {
                    return CountryData($route.current.params.id);
                }]
            }
        })
    }])

An easy test would be:

    describe('country.js', function(){
        beforeEach(module('cc-app'));
        
        it('should load the correct template and controller', function(){
            inject(function($route){
                expect($route.routes['/countries/:id'].controller).toBe('countryCtrl');
                expect($route.routes['/countries/:id'].templateUrl).toBe('./country/country.html');
            });
        });
    });
    
This works, but it's not very good. If the app is reorganized, then any test like this will have to change because the paths in the `expect` won't match.

This test also isn't really testing anything other than the configurations - it doesn't load the route in the test block. A better test would actually load the route:

    it('should load the correct template and controller', function(){
        inject(function($httpBackend, $location, $rootScope, $route){
            var id = 'FR';
        
            $httpBackend.expectGET('/country.html').respond('...');
      
            $rootScope.$apply(function(){
                $location.path('/countries/' + id);
            });
      
            $httpBackend.flush();
            $httpBackend.verifyNoOutstandingRequest();
            expect($route.current.controller).toBe('countryCtrl');
            expect($route.current.templateUrl).toBe('./country/country.html');
        });
    });

This is nicer since it actually instantiates the route. We could also use a regex for the `templateUrl` to make the code easier to change later.

But this test has an asynchronous request problem: the service in the `resolve` block never resolves, so the will fail with an `unexpected request` from the route `resolve`. We could use `$httpBackend` and `done` like in the previous example, but since we've already written a test for that service, does it really need to be tested again? And does this routing test even need to know that `resolve` calls an asynchronous function? An alternative would be to double it out with a `spy`.

http://jasmine.github.io/2.0/introduction.html#section-Spies

A spy is a double of a function. Instead of calling a function, the test will call the spy which can be created to behave like the function it replaces, but without the asynchronous call. In jasmine, there are matchers to confirm that the spy is behaving as it should.

In Jasmine spies are objects and so to properly double a function, it needs to be the method of some object:

    var fakeData = {
        stuff: function(arg){
        return arg;
        }
    }
    
This is just a plain old javascript object: `fakeData.stuff('whatever');`

The spy is setup like this: `spyOn(fakeData, 'stuff');`. `spyOn` takes two arguments: the object and one of its properties. All this means is that jasmine is now paying attention to `fakeData.stuff`. The test now has access to the `toHaveBeenCalled()` matcher that verifies that the function was actually called in the test block.

The next step is to tell angular to use the spy in place of the function it doubles. `$provide` is how angular registers components to inject them into a module, so the code:

    module(function($provide){
        $provide.factory('CountryData', function(){
            return fakeData.stuff;
        });
    });
    
tells angular to use `fakeData.stuff` when the `CountryData` factory is called. Remember the `resolve` in the controller?

    resolve: {
        ActiveCountry: ['CountryData', '$route', function(CountryData, $route) {
            return CountryData($route.current.params.id);
        }]
    }
    
This resolve needs a function that takes one argument, which is exactly what `fakeData.stuff` will do. The return value of this factory doesn't matter so much since we're only testing the route: it's the controller's job to set `ActiveCountry` to the appropriate `scope`.

The full test looks like this:

    describe('country.js', function(){
        
        var fakeData = {
            stuff: function(arg){
                return arg;
            }
        }
        
        beforeEach(module('cc-app'));
        
        beforeEach(function(){
        
            spyOn(fakeData, 'stuff');
        
            module(function($provide){
                $provide.factory('CountryData', function(){
                    return fakeData.stuff;
                });
            });
        });
        
        it('should load the correct template and controller', function(){
            inject(function($httpBackend, $location, $rootScope, $route){
        
                var id = 'FR';
        
                $httpBackend.expectGET('./country/country.html').respond('...');
        
                $rootScope.$apply(function(){
                    $location.path('/countries/' + id);
                });
      
                $httpBackend.flush();
                $httpBackend.verifyNoOutstandingRequest();
        
                expect($route.current.controller).toBe('countryCtrl');
                expect($route.current.templateUrl).toBe('./country/country.html');
                expect(fakeData.stuff).toHaveBeenCalledWith(id);
                }); 
            });
        });
        
Notice that instead of just `expect(fakeData.stuff).toHaveBeenCalled();` there is another matcher `toHaveBeenCalled()` that can verify that the correct arguments were also passed.

The first routing test only verified that the router was configured correctly. This much-improved version verifies that given the url `/countries/:id`, the router loads the correct controller and template AND passes the correct params from the url to `resolve` which then runs some function that takes those params as an argument. 