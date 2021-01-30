# Writing Controllers 


### Guidelines ::

When you're writing controllers, there are few guidelines that will help make sure you get the results and performance you're looking for.

1. Operate on one item at a time.  If you use a `workqueue.Interface`, you'll be able to queue changes for a particular resource and later pop them in multiple “worker” gofuncs with a guarantee that no two gofuncs will work on the same item at the same time.

   Many controllers must trigger off multiple resources (I need to "check X if Y changes"), but nearly all controllers can collapse those into a queue of “check this X” based on relationships.  For instance, a ReplicaSet controller needs to react to a pod being deleted, but it does that by finding the related ReplicaSets and queuing those.

1. Random ordering between resources. When controllers queue off multiple types of resources, there is no guarantee of ordering amongst those resources.

   Distinct watches are updated independently.  Even with an objective ordering of “created resourceA/X” and “created resourceB/Y”, your controller could observe “created resourceB/Y” and “created resourceA/X”.

1. Level driven, not edge driven.  Just like having a shell script that isn't running all the time, your controller may be off for an indeterminate amount of time before running again.

   If an API object appears with a marker value of `true`, you can't count on having seen it turn from `false` to `true`, only that you now observe it being `true`.  Even an API watch suffers from this problem, so be sure that you're not counting on seeing a change unless your controller is also marking the information it last made the decision on in the object's status.

1. Use `SharedInformers`.  `SharedInformers` provide hooks to receive notifications of adds, updates, and deletes for a particular resource.  They also provide convenience functions for accessing shared caches and determining when a cache is primed.

   Use the factory methods down in https://git.k8s.io/kubernetes/staging/src/k8s.io/client-go/informers/factory.go to ensure that you are sharing the same instance of the cache as everyone else.

   This saves us connections against the API server, duplicate serialization costs server-side, duplicate deserialization costs controller-side, and duplicate caching costs controller-side.

   You may see other mechanisms like reflectors and deltafifos driving controllers.  Those were older mechanisms that we later used to build the `SharedInformers`.  You should avoid using them in new controllers.

1. Never mutate original objects!  Caches are shared across controllers, this means that if you mutate your "copy" (actually a reference or shallow copy) of an object, you'll mess up other controllers (not just your own).

   The most common point of failure is making a shallow copy, then mutating a map, like `Annotations`.  Use `api.Scheme.Copy` to make a deep copy. (https://www.educative.io/edpresso/deep-copying-vs-shallow-copying)

1. Wait for your secondary caches.  Many controllers have primary and secondary resources.  Primary resources are the resources that you'll be updating `Status` for.  Secondary resources are resources that you'll be managing (creating/deleting) or using for lookups.

   Use the `framework.WaitForCacheSync` function to wait for your secondary caches before starting your primary sync functions.  This will make sure that things like a Pod count for a ReplicaSet isn't working off of known out of date information that results in thrashing.

1. There are other actors in the system.  Just because you haven't changed an object doesn't mean that somebody else hasn't.

   Don't forget that the current state may change at any moment--it's not sufficient to just watch the desired state. If you use the absence of objects in the desired state to indicate that things in the current state should be deleted, make sure you don't have a bug in your observation code (e.g., act before your cache has filled).

1. Percolate errors to the top level for consistent re-queuing.  We have a  `workqueue.RateLimitingInterface` to allow simple requeuing with reasonable backoffs.

   Your main controller func should return an error when requeuing is necessary.  When it isn't, it should use `utilruntime.HandleError` and return nil instead.  This makes it very easy for reviewers to inspect error handling cases and to be confident that your controller doesn't accidentally lose things it should retry for.

1. Watches and Informers will “sync”.  Periodically, they will deliver every matching object in the cluster to your `Update` method.  This is good for cases where you may need to take additional action on the object, but sometimes you know there won't be more work to do.

   In cases where you are *certain* that you don't need to requeue items when there are no new changes, you can compare the resource version of the old and new objects.  If they are the same, you skip requeuing the work.  Be careful when you do this.  If you ever skip requeuing your item on failures, you could fail, not requeue, and then never retry that item again.

1. If the primary resource your controller is reconciling supports ObservedGeneration in its status, make sure you correctly set it to metadata.Generation whenever the values between the two fields mismatches.

    This lets clients know that the controller has processed a resource. Make sure that your controller is the main controller that is responsible for that resource, otherwise if you need to communicate observation via your own controller, you will need to create a different kind of ObservedGeneration in the Status of the resource.

1. Consider using owner references for resources that result in the creation of other resources (eg. a ReplicaSet results in creating Pods). Thus you ensure that children resources are going to be garbage-collected once a resource managed by your controller is deleted. For more information on owner references, read more [here](/contributors/design-proposals/api-machinery/controller-ref.md).

    Pay special attention in the way you are doing adoption. You shouldn't adopt children for a resource when either the parent or the children are marked for deletion. If you are using a cache for your resources, you will likely need to bypass it with a direct API read in case you observe that an owner reference has been updated for one of the children. Thus, you ensure your controller is not racing with the garbage collector.
