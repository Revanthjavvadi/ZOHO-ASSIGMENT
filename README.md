# Countact-Manager-App


{
  "name": "contactmanager",
  "version": "1.0.0",
  "description": "Contact Manager App developed with MEAN Stack",
  "main": "server.js",
  "dependencies": {
    "body-parser": "^1.10.2",
    "express": "^4.11.1",
    "mongojs": "^0.18.1"
  },
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1",
    "start": "node server.js"
  },
  "repository": {
    "type": "git",
    "url": "git+https://github.com/hherzl/ContactManager.Mean.git"
  },
  "author": "C. Herzl",
  "license": "ISC",
  "bugs": {
    "url": "https://github.com/hherzl/ContactManager.Mean/issues"
  },
  "homepage": "https://github.com/hherzl/ContactManager.Mean#readme"
}
var express = require("express");
var bodyParser = require("body-parser");

var app = express();
app.use(express.static(__dirname + "/public"));
app.use(bodyParser.json());

var rest = require("./api");
var api = new rest(app);

var port = 3000;

app.listen(port);

console.log("Server running on port " + port);
function api (app) {
    var mongojs = require("mongojs");
    
    var db = mongojs("contactManager", ["contacts"]);

    app.get("/api/contact", function (request, response) {
        var pageSize = request.query.pageSize ? parseInt(request.query.pageSize) : 10;
        var firstName = request.query.firstName;
        var middleName = request.query.middleName;
        var lastName = request.query.lastName;

        var find = {};

        if (firstName) {
            find.firstName = new RegExp(firstName, "i");
        }

        if (middleName) {
            find.middleName = new RegExp(middleName, "i");
        }

        if (lastName) {
            find.lastName = new RegExp(lastName, "i");
        }

        var fields = {
            firstName: 1,
            middleName: 1,
            lastName: 1,
            phone: 1,
            email: 1
        };

        var result = db.contacts.find(find, fields).limit(pageSize, function (err, docs) {
            response.json(docs);
        });
    });

    app.get("/api/contact/:id", function (request, response) {
        var id = request.params.id;

        db.contacts.findOne({ _id: mongojs.ObjectId(id) }, function (err, doc) {
            if (err)
                console.log("Error: " + err);

            response.json(doc);
        });
    });

    app.post("/api/contact", function (request, response) {
        db.contacts.insert(request.body, function (err, doc) {
            if (err)
                console.log("Error: " + err);

            response.json(doc);
        });
    });

    app.put("/api/contact/:id", function (request, response) {
        var id = request.params.id;

        db.contacts.findAndModify({
            query: {
                _id: mongojs.ObjectId(id)
            },
            update: {
                $set: {
                    firstName: request.body.firstName,
                    middleName: request.body.middleName,
                    lastName: request.body.lastName,
                    phone: request.body.phone,
                    email: request.body.email
                }
            },
            new: true
        }, function (err, doc) {
            response.json(doc);
        });
    });

    app.delete("/api/contact/:id", function (request, response) {
        var id = request.params.id;

        db.contacts.remove({ _id: mongojs.ObjectId(id) }, function (err, doc) {
            if (err)
                console.log("Error: " + err);

            response.json(doc);
        });
    });
};

module.exports = api;
var module = angular.module("contactManager", [
    "ngRoute",
    "ngAnimate",
    "toaster"
]);

(function (app) {
    app.config(function ($routeProvider) {
        var base = "/views/";

        $routeProvider
            .when("/", {
                templateUrl: base + "contact/index.html",
                controller: "HomeController",
                controllerAs: "vm"
            })
            .when("/contact/add", {
                templateUrl: base + "contact/add.html",
                controller: "ContactAddController",
                controllerAs: "vm"
            })
            .when("/contact/details/:id", {
                templateUrl: base + "contact/details.html",
                controller: "ContactDetailsController",
                controllerAs: "vm"
            })
            .when("/contact/edit/:id", {
                templateUrl: base + "contact/edit.html",
                controller: "ContactEditController",
                controllerAs: "vm"
            })
            .when("/contact/remove/:id", {
                templateUrl: base + "contact/remove.html",
                controller: "ContactRemoveController",
                controllerAs: "vm"
            });
    });
})(angular.module("contactManager"));
(function (app) {
    "use strict";

    app.service("RepositoryService", RepositoryService);

    RepositoryService.$inject = ["$log", "$http"];

    function RepositoryService($log, $http) {
        var svc = this;

        var apiUrl = "/api";

        svc.getContacts = getContacts;
        svc.getContact = getContact;
        svc.createContact = createContact;
        svc.updateContact = updateContact;
        svc.deleteContact = deleteContact;

        function getContacts(fields) {
            var queryString = [];

            if (fields.pageSize) {
                queryString.push("pageSize=" + fields.pageSize);
            }

            if (fields.firstName) {
                queryString.push("firstName=" + fields.firstName);
            }

            if (fields.middleName) {
                queryString.push("middleName=" + fields.middleName);
            }

            if (fields.lastName) {
                queryString.push("lastName=" + fields.lastName);
            }

            var url = [apiUrl, "contact"].join("/");

            var fullUrl = queryString.length == 0 ? url : [url, "?", queryString.join("&")].join("");

            return $http.get(fullUrl);
        };

        function getContact(id) {
            return $http.get([apiUrl, "contact", id].join("/"));
        };

        function createContact(model) {
            return $http.post([apiUrl, "contact"].join("/"), model);
        };

        function updateContact(id, model) {
            return $http.put([apiUrl, "contact", id].join("/"), model);
        };

        function deleteContact(id) {
            return $http.delete([apiUrl, "contact", id].join("/"));
        };
    };
})(angular.module("contactManager"));
(function (app) {
    "use strict";

    app.controller("HomeController", HomeController);

    HomeController.$inject = ["$location", "toaster", "RepositoryService"];

    function HomeController($location, toaster, repository) {
        var vm = this;

        vm.contacts = [];
        vm.search = {};
        vm.add = add;
        vm.search = search;
        vm.details = details;
        vm.remove = remove;

        toaster.pop("wait", "Loading contacts...");

        repository.getContacts(vm.search).then(function (result) {
            vm.contacts = result.data;
        });

        vm.add = function () {
            $location.path("/contact/add/");
        };

        vm.search = function () {
            repository.getContacts(vm.search).then(function (result) {
                vm.contacts = result.data;
            });
        };
        
        vm.details = function (id) {
            $location.path("/contact/details/" + id);
        };

        vm.remove = function (id) {
            $location.path("/contact/remove/" + id);
        };
    };
})(angular.module("contactManager"));
<h2>Contacts</h2>

<p>
    <a ng-click="vm.add()">Add new</a>
</p>

<div class="container">
    <div class="row">
        <div class="form-inline">
            <div class="form-group">
                <input type="text" name="firstName" class="form-control" placeholder="First name" ng-model="vm.search.firstName" />
            </div>

            <div class="form-group">
                <input type="text" name="middleName" class="form-control" placeholder="Middle name" ng-model="vm.search.middleName" />
            </div>

            <div class="form-group">
                <input type="text" name="lastName" class="form-control" placeholder="Last name" ng-model="vm.search.lastName" />
            </div>

            <div class="form-group">
                <select name="pageSize" class="form-control list-box" ng-model="vm.search.pageSize">
                    <option value="10" selected="selected">10</option>
                    <option value="25">25</option>
                    <option value="50">50</option>
                    <option value="100">100</option>
                </select>
            </div>

            <div class="form-group">
                <button type="button" class="btn btn-primary glyphicon glyphicon-search" ng-click="vm.search();" />
            </div>
        </div>
    </div>
</div>

<br />

<table class="table table-hover">
    <tr>
        <th>First name</th>
        <th>Middle name</th>
        <th>Last name</th>
        <th>Phone</th>
        <th>Email</th>
        <th></th>
    </tr>
    <tr ng-repeat="contact in vm.contacts">
        <td>{{ contact.firstName }}</td>
        <td>{{ contact.middleName }}</td>
        <td>{{ contact.lastName }}</td>
        <td>{{ contact.phone }}</td>
        <td>{{ contact.email }}</td>
        <td>
            <div class="dropdown">
                <button class="btn btn-primary btn-lg dropdown-toggle" type="button" data-toggle="dropdown">
                <span class="caret"></span></button>
                <ul class="dropdown-menu">
                    <li><a ng-click="vm.details(contact._id)">Details</a></li>
                    <li><a ng-click="vm.remove(contact._id)">Remove</a></li>
                </ul>
            </div>
        </td>
    </tr>
</table>
(function (app) {
    "use strict";

    app.controller("ContactAddController", ContactAddController);

    ContactAddController.$inject = ["$location", "toaster", "RepositoryService"];

    function ContactAddController($location, toaster, repository) {
        var vm = this;

        vm.model = {};

        vm.save = save;
        vm.cancel = cancel;

        function save() {
            toaster.pop("wait", "Saving...");

            repository.createContact(vm.model).then(function (result) {
                toaster.pop("success", "The contact was saved successfully");

                $location.path("/");
            });
        };

        function cancel() {
            $location.path("/");
        };
    };
})(angular.module("contactManager"));
<h2>Create</h2>

<form name="form" novalidate ng-submit="vm.save()">
    <div class="form-horizontal">
        <h4>Contact</h4>
        <hr />

        <div class="form-group" ng-class="{ 'has-error': form.firstName.$invalid }">
            <label for="firstName" class="control-label col-md-2">First name</label>
            <div class="col-md-10">
                <input type="text" name="firstName" class="form-control" required ng-model="vm.model.firstName" />
                <span class="text-danger help-block" ng-show="form.firstName.$error.required">(*) Required</span>
            </div>
        </div>

        <div class="form-group">
            <label for="middleName" class="control-label col-md-2">Middle name</label>
            <div class="col-md-10">
                <input type="text" name="middleName" class="form-control" ng-model="vm.model.middleName" />
            </div>
        </div>

        <div class="form-group" ng-class="{ 'has-error': form.lastName.$invalid }">
            <label for="lastName" class="control-label col-md-2">Last name</label>
            <div class="col-md-10">
                <input type="text" name="lastName" class="form-control" required ng-model="vm.model.lastName" />
                <span class="text-danger help-block" ng-show="form.lastName.$error.required">(*) Required</span>
            </div>
        </div>

        <div class="form-group" ng-class="{ 'has-error': form.phone.$invalid }">
            <label for="phone" class="control-label col-md-2">Phone</label>
            <div class="col-md-10">
                <input type="text" name="phone" class="form-control" required ng-model="vm.model.phone" />
                <span class="text-danger help-block" ng-show="form.phone.$error.required">(*) Required</span>
            </div>
        </div>

        <div class="form-group" ng-class="{ 'has-error': form.email.$invalid }">
            <label for="email" class="control-label col-md-2">Email</label>
            <div class="col-md-10">
                <input type="text" name="email" class="form-control" required ng-model="vm.model.email" />
                <span class="text-danger help-block" ng-show="form.email.$error.required">(*) Required</span>
            </div>
        </div>

        <div class="form-group">
            <div class="col-md-offset-2 col-md-10">
                <input type="submit" value="Save" ng-disabled="!form.$valid" class="btn btn-primary" />
                <a ng-click="vm.cancel()">Cancel</a>
            </div>
        </div>
    </div>
</form>
(function (app) {
    "use strict";

    app.controller("ContactDetailsController", ContactDetailsController);

    ContactDetailsController.$inject = ["$routeParams", "$location", "RepositoryService"];

    function ContactDetailsController($routeParams, $location, repository) {
        var vm = this;

        var id = $routeParams.id;

        vm.model = {};
        vm.edit = edit;
        vm.cancel = cancel;

        repository.getContact(id).then(function (result) {
            vm.model = result.data;
        });

        function edit() {
            $location.path("/contact/edit/" + id);
        };

        function cancel() {
            $location.path("/");
        };
    };
})(angular.module("contactManager"));
<h2>Details</h2>

<div>
    <h4>Contact</h4>
    <hr />
    <dl class="dl-horizontal">
        <dt>First name</dt>
        <dd>{{ vm.model.firstName }}</dd>

        <dt>Middle name</dt>
        <dd>{{ vm.model.middleName }}</dd>

        <dt>Last name</dt>
        <dd>{{ vm.model.lastName }}</dd>

        <dt>Phone</dt>
        <dd>{{ vm.model.phone }}</dd>

        <dt>Email</dt>
        <dd>{{ vm.model.email }}</dd>
    </dl>
</div>

<p>
    <a ng-click="vm.edit()">Edit</a> | <a ng-click="vm.cancel()">Cancel</a>
</p>
(function (app) {
    "use strict";

    app.controller("ContactEditController", ContactEditController);

    ContactEditController.$inject = ["$routeParams", "$location", "toaster", "RepositoryService"];

    function ContactEditController($routeParams, $location, toaster, repository) {
        var vm = this;

        var id = $routeParams.id;

        vm.model = {};
        vm.save = save;
        vm.cancel = cancel;

        repository.getContact(id).then(function (result) {
            vm.model = result.data;
        });

        function save() {
            toaster.pop("wait", "Saving...");

            repository.updateContact(id, vm.model).then(function (result) {
                toaster.pop("success", "The changes were saved successfully");

                $location.path("/contact/details/" + id);
            });
        };

        function cancel() {
            $location.path("/contact/details/" + id);
        };
    };
})(angular.module("contactManager"));
<h2>Edit</h2>

<form name="form" novalidate ng-submit="vm.save()">
    <div class="form-horizontal">
        <h4>Contact</h4>
        <hr />

        <div class="form-group" ng-class="{ 'has-error': form.firstName.$invalid }">
            <label for="firstName" class="control-label col-md-2">First name</label>
            <div class="col-md-10">
                <input type="text" name="firstName" class="form-control" required ng-model="vm.model.firstName" />
                <span class="text-danger help-block" ng-show="form.firstName.$error.required">(*) Required</span>
            </div>
        </div>

        <div class="form-group">
            <label for="middleName" class="control-label col-md-2">Middle name</label>
            <div class="col-md-10">
                <input type="text" name="middleName" class="form-control" ng-model="vm.model.middleName" />
            </div>
        </div>

        <div class="form-group" ng-class="{ 'has-error': form.lastName.$invalid }">
            <label for="lastName" class="control-label col-md-2">Last name</label>
            <div class="col-md-10">
                <input type="text" name="lastName" class="form-control" required ng-model="vm.model.lastName" />
                <span class="text-danger help-block" ng-show="form.lastName.$error.required">(*) Required</span>
            </div>
        </div>

        <div class="form-group" ng-class="{ 'has-error': form.phone.$invalid }">
            <label for="phone" class="control-label col-md-2">Phone</label>
            <div class="col-md-10">
                <input type="text" name="phone" class="form-control" required ng-model="vm.model.phone" />
                <span class="text-danger help-block" ng-show="form.phone.$error.required">(*) Required</span>
            </div>
        </div>

        <div class="form-group" ng-class="{ 'has-error': form.email.$invalid }">
            <label for="email" class="control-label col-md-2">Email</label>
            <div class="col-md-10">
                <input type="text" name="email" class="form-control" required ng-model="vm.model.email" />
                <span class="text-danger help-block" ng-show="form.email.$error.required">(*) Required</span>
            </div>
        </div>

        <div class="form-group">
            <div class="col-md-offset-2 col-md-10">
                <input type="submit" value="Save" ng-disabled="!form.$valid" class="btn btn-primary" />
                <a ng-click="vm.cancel()">Cancel</a>
            </div>
        </div>
    </div>
</form>
(function (app) {
    "use strict";

    app.controller("ContactRemoveController", ContactRemoveController);

    ContactRemoveController.$inject = ["$location", "$routeParams", "toaster", "RepositoryService"];

    function ContactRemoveController($location, $routeParams, toaster, repository) {
        var vm = this;

        var id = $routeParams.id;

        vm.model = {};
        vm.remove = remove;
        vm.cancel = cancel;

        repository.getContact(id).then(function (result) {
            vm.model = result.data;
        });

        function remove() {
            toaster.pop("wait", "Removing...");

            repository.deleteContact(id).then(function (result) {
                toaster.pop("success", "The contact was removed successfully");

                $location.path("/");
            });
        };

        function cancel() {
            $location.path("/");
        };
    };
})(angular.module("contactManager"));
<h2>Remove</h2>

<h3>Are you sure to want remove this record?</h3>

<div>
    <h4>Contact</h4>
    <hr />
    <dl class="dl-horizontal">
        <dt>First name</dt>
        <dd>{{ vm.model.firstName }}</dd>

        <dt>Middle name</dt>
        <dd>{{ vm.model.middleName }}</dd>

        <dt>Last name</dt>
        <dd>{{ vm.model.lastName }}</dd>

        <dt>Phone</dt>
        <dd>{{ vm.model.phone }}</dd>

        <dt>Email</dt>
        <dd>{{ vm.model.email }}</dd>
    </dl>

    <div class="form-actions no-color">
        <a ng-click="vm.remove(vm.model._id)" class="btn btn-default">Delete</a>
        <a ng-click="vm.cancel()" class="btn btn-small">Cancel</a>
    </div>
</div>var mongojs = require("mongojs");
var db = mongojs("contactManager", ["contacts"]);

var limit = 1000;

var firstNames = ["James", "John", "Robert", "Michael", "William", "David", "Richard", "Charles", "Joseph", "Thomas"];

var middleNames = ["Peter", "Lee", "Alexander", "Daniel", "Edward"];

var lastNames = ["Smith", "Johnson", "Williams", "Jones", "Brown", "Davis", "Miller", "Wilson", "Moore"];

var getRandomValue = function (min, max) {
    return Math.floor(Math.random() * ((max - min) + 1)) + min;
};

var getRandomItem = function (array) {
    return array[Math.floor(Math.random() * array.length)];
};

var phoneMocker = function () {
    var areaCode = getRandomValue(250, 500);
    var phoneNumber = getRandomValue(2000000, 5000000);

    return [areaCode, " ", phoneNumber].join("");
};

var emailMocker = function (contact) {
    var separators = ["", ".", "_", "", ".", "_"];
    var separator = getRandomItem(separators);

    var contactInfo = [contact.firstName, separator, contact.middleName, separator, contact.lastName].join("");

    var domains = ["outlook.com", "gmail.com", "yahoo.com"];

    return contactInfo.toLowerCase() + "@" + getRandomItem(domains);
};

for (var i = 0; i < limit; i++) {
    var firstName = getRandomItem(firstNames);
    var middleName = getRandomItem(middleNames);
    var lastName = getRandomItem(lastNames);

    var item = {
        firstName: firstName,
        middleName: middleName,
        lastName: lastName,
        phone: phoneMocker()
    };

    item.email = emailMocker(item);

    console.log("Inserting new row...");

    db.contacts.insert(item, function (err, doc) {
        if (err) {
            console.log(err);
        }
    });
}
