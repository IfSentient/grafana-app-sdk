# Adding Admission Control

One thing we can do with our operator is add in _admission control_. In the grafana-app-sdk, admission control refers to performing validation and/or mutation on incoming requests to the API server. Since our operator doesn't receive these requests, we have to expose a webhook for the API server to call when it receives a request, and then register that webhook with the API server. Luckily, this is not quite so complicated as it sounds. Let's add some validation and some mutation to our `Issue` kind using our operator. We want to do the following when an `Issue` is created or updated:
1. Check that the `title` isn't empty
2. Add a `status` label that matches the `spec.status`, to allow us to filter our list requests by `status`
For (1), we need to perform validation, and for (2), we need to do a mutation. So lets add validating and mutating controllers to our operator.

## Validation

Luckily, with the `simple.Operator` we're using that was generated by `component add operator`, adding validation and mutation for a kind is relatively simple. Let's add the following after line 75 (after the `WatchKind` handling, but before we create `stopCh`):
```go
err = runner.ValidateKind(issue.Kind(), &resource.SimpleValidatingAdmissionController{
    ValidateFunc: func(ctx context.Context, request *resource.AdmissionRequest) error {
        cast, ok := request.Object.(*issue.Issue)
        if !ok {
            return fmt.Errorf("object is not of type *issue.Issue (%s %s)", request.Object.GetName(), request.Object.GroupVersionKind().String())
        }
        if strings.Trim(cast.Spec.Title, " ") == "" {
            return fmt.Errorf("spec.title must not be empty or consist only of whitespace characters")
        }
        return nil
    },
})
if err != nil {
    panic(err)
}
```
Just like with `runner.WatchKind`, `runner.ValidateKind` takes the kind, and then the object to apply to it. In this case, to validate a kind, we need a `ValidatingAdmissionController`. This is a simple one-method interface, which we could define a type for ourselves, but we can also use `resource.SimpleValidatingAdmissionController` as a default implementation. In `SimpleValidatingAdmissionController`, `ValidateFunc` is called by the `Validate` function, so we just need to define our validation function in `ValidateFunc`.

`Validate` receives a context, and a request or type `*resource.AdmissionRequest`. The request contains some meta information that we can ignore for now (including action type, user information, and GroupVersionKind), and the `Object` from the request (and an `OldObject` which may be nil if there was not a previous state of the object, depending on the action). We can use this `Object` value to check if the updated or created object has an empty title, and return an error. An important thing to note is that  text of the `error` returned by `Validate` will be returned to the end-user, so make sure it's as descriptive as necessary for the user to fix their request. If no error is returned by `Validate`, the validation passes.

If we re-built our operator now and attempted to run it locally, our validation wouldn't occur, and we'd actually get an error: `webhooks are not enabled`. That's because we didn't enable webhooks in our config in `simple.NewOperator`. So let's do that. In the bottom of our `simple.OperatorConfig` block, add the following:
```go
Webhooks: simple.WebhookConfig{
    Enabled: true,
    Port:    cfg.WebhookServer.Port,
    TLSConfig: k8s.TLSConfig{
        CertPath: cfg.WebhookServer.TLSCertPath,
        KeyPath:  cfg.WebhookServer.TLSKeyPath,
    },
},
```
This allows the operator to expose the webhook HTTPS server, using the port and TLS config provided. Now, we just need to make sure those config values are populated. Luckily, in a local setup, this is done automatically if `webhooks.validating` (or `webhooks.mutating` or `webhooks.converting`) is `true` in `local/config.yaml`. This will also generate the appropriate kubernetes manifests to register the validating webhook, and create the cert bundle. So now, if we set `webhooks.validating` to `true` in `local/config.yaml`, and then build our operator and re-run `make local/up`:
```shell
make local/down && make build/operator && make local/push_operator && make local/up
```
Now, if we attempt to make an issue with an empty Title (or one only consisting of spaces), we get an error:
```
admission webhook "issue-tracker-project-app-operator.default.svc" denied the request: spec.title must not be empty or consist only of whitespace characters
```
But, if we add any non-whitespace characters, we see that we can still successfull make the `issue`.

## Mutation

Next, we want to make sure that there is a label named `status` that always matches `spec.status` for each object. We can do this with a `MutatingAdmissionController`. Let's add the following below our `runner.ValidateKind` block:
```go
err = runner.MutateKind(issue.Kind(), &resource.SimpleMutatingAdmissionController{
    MutateFunc: func(ctx context.Context, request *resource.AdmissionRequest) (*resource.MutatingResponse, error) {
        cast, ok := request.Object.(*issue.Issue)
        if !ok {
            return nil, fmt.Errorf("object is not of type *issue.Issue (%s %s)", request.Object.GetName(), request.Object.GroupVersionKind().String())
        }
        if cast.Labels == nil {
            cast.Labels = make(map[string]string)
        }
        cast.Labels["status"] = cast.Spec.Status
        return &resource.MutatingResponse{
            UpdatedObject: cast,
        }, nil
    },
})
if err != nil {
    panic(err)
}
```
We add a `MutatingAdmissionController` much like we do a validating one. And much like the validating one, there's a default implementation we can use where we just fill out `MutateFunc`. However, notice that `Mutate` accepts the same arguments are `Validate`, but returns a `*resource.MutatingResponse` in addition to an `error`. This MutatingResponse is used to pass along the updated object, which is then used as the version to put into storage (after validation occurs).

Now, all we need to do is remember to set `webhooks.mutating` to `true` in our `local/config.yaml`, and then re-run:
```shell
make local/down && make build/operator && make local/push_operator && make local/up
```
Now, whenever you make or update an issue, you'll see that it has a label attached for its status. You can see these in the network tab of your browser console, or via `kubectl`. In fact, you can now filter issues using kubectl:
```
kubectl get issues -l status=open
```
Will give you only issues with a status of `open`, for example.

The neat part about this validation and mutation is that it occurs irrespective of how the user decides to create or update their issues. Since the webhooks are called by the API server, if the user uses `kubectl`, `flux`, or the plugin UI, they all will route to the same API server backend, which will call our validating and mutating webhooks.

For more details on webhooks and admission control, see [Admission Control](../../admission-control.md).

### Prev: [Writing Operator Code](07-operator-watcher.md)
### Next: [Wrap-Up and Further Reading](09-wrap-up.md)