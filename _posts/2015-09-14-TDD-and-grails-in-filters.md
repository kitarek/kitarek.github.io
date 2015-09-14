---
layout: post
category : grails
tagline: "Effective TDD in Grails filters"
tags : [groovy, TDD, grails]
---
{% include JB/setup %}


## Request filters in Grails

In Grails there is a special construction for implementing any request filters.
The class must accordingly names with `Filters` suffix and there is a special
closure filters which contains different definitions of menu entry points and
even different implementations for filters. That's why the class name is
`Filters` and not one `Filter`.


    class ExampleFilters {
        def filters = {
            allExceptIndex(controller: "site", action: "index", invert: true) {
                before = { }
                after = { Map model ->  }
                afterView = { Exception e -> }
            },
            siteOnly(controller: "site") {
                before = { }
                after = { Map model ->  }
                afterView = { Exception e -> }
            }
        }
    }


## Filtering I was doing...

In general Grails 2 filters can do many things, but I had to implement sth that
logs out any fraud user, and then don't allow to log in back at anytime.

The simplest thing you can think of is:

    class FraudUserFilters {

       def filters = {
           all(controller: 'auth', invert: true) {
                before = {
                    def user = User.get(request.session.myAppSession.userId)
                    if (isFraudUser(user)) {
                        log.info("Signing out fraud user: ${user.email} (${user.id})");
                        redirect(controller: "auth", action: "signOut")
                    }
                }
            }
        }
        ...
    }

Obviously the `isFraudUser` method can be easily tested with any Spock/JUnit
but there is no way to write correct unit tests for the rest of the filter which
stay in that closure.

The best would be to extract the whole content of filter into separate method
like this:


    class FraudUserFilters {

       def filters = {
           all(controller: 'auth', invert: true) {
                before = {
                    onBefore()
                }
            }
        }

        def onBefore() {
            def user = User.get(request.session.myAppSession.userId)
            if (isFraudUser(user)) {
                log.info("Signing out fraud user: ${user.email} (${user.id})");
                redirect(controller: "auth", action: "signOut")
            }
        }

        ...
    }

The problem is that `redirect` operation is not available in any method outside
of filter interceptor closures since it is a bit different scope that is hidden
here.

In general it is both hard to write this properly and hard to test.

## The best solution I found so far

In that case instead of method we can use closure that will be defined in
scope of that interceptor:


       def filters = {
           all(controller: 'auth', invert: true) {
                before = {
                    def redirectToControllerAndAction = { controller, action ->
                        redirect(controller: controller, action: action)
                    }
                    onBefore(redirectToControllerAndAction)
                }
            }
        }

This way we have the whole logic exported to method and the internal redirect
method which is normally not available outside of filter is wrapped with closure.

Moreover that closure is now explicitly defined as a one of input argument so
in other words we are passing message to input argument in case we would like
to do any redirects.

This solution is now very easy to test because we can define input closure as
spy and check whether it is called at all and with with arguments it is invoked.

So the full implementation is now:

    class FraudUserFilters {

       def filters = {
           all(controller: 'auth', invert: true) {
                before = {
                    def redirectToControllerAndAction = { controller, action ->
                        redirect(controller: controller, action: action)
                    }

                    redirectWhenCurrentUserIsBlocked(redirectToControllerAndAction)
                }
            }
        }

        def redirectWhenCurrentUserIsBlocked(Closure redirectToControllerAndAction) {
            def user = User.get(request.session.myAppSession.userId)
            if (isFraudUser(user)) {
                log.info("Signing out fraud user: ${user.email} (${user.id})");
                redirectToControllerAndAction("auth", "signOut")
            }
        }

        ...
    }

This way the whole critical code can be unit tested and as the only one filter
dependency is required function argument.

Furthermore TDD of such filter doesn't require any special procedure. You can
even skip `filters` section and develop the whole filter logic and then add it
testing end to end.

