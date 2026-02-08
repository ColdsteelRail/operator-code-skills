---
name: k8s-controller-integration-test
description: Create comprehensive integration tests for Kubernetes controllers using envtest + ginkgo + gomega. Use when the task involves (1) Writing integration tests for Kubernetes controllers/reconcilers, (2) Setting up test environment with envtest, (3) Writing controller logic tests that verify resource creation/updates/deletion, (4) Testing controller-manager integration with CRDs, or any task related to Kubernetes controller testing.
---

# K8s Controller Integration Test

## Overview

Create integration tests for Kubernetes controllers using envtest, Ginkgo, and Gomega. These tests run against a local Kubernetes-like control plane and verify end-to-end controller behavior including resource creation, status reconciliation, finalizer handling, and owner reference management.

## Quick Start

Initialize a new controller integration test:

```go
package mycontroller

import (
    "context"
    "path/filepath"
    "testing"
    "time"

    . "github.com/onsi/ginkgo"
    . "github.com/onsi/gomega"
    ...
)

func TestMyControllerWithSetupWithManager(t *testing.T) {
    RegisterFailHandler(Fail)
    RunSpecs(t, "My Controller Suite")
}

// Global variables
var (
    testEnv   *envtest.Environment
    k8sClient client.Client
    ctx       context.Context
    cancel    context.CancelFunc
)
```

## Test Structure

```
{controller}_controller_test.go
├── BeforeSuite   // Initialize envtest, config, manager, controller
├── AfterSuite    // Tear down envtest
├── AfterEach     // Clean up resources and namespaces
└── Test specs    // Actual test cases
```

## BeforeSuite Template

```go
var _ = BeforeSuite(func() {
    // 1. Set up logging
    loggerWriter := zapcore.NewMultiWriteSyncer(
        zapcore.AddSync(GinkgoWriter),
        zapcore.AddSync(os.Stdout),
    )
    logger := zap.New(zap.WriteTo(loggerWriter), zap.UseDevMode(true))
    logf.SetLogger(logger)

    ctx, cancel = context.WithCancel(context.Background())

    // 2. Start envtest
    testEnv = &envtest.Environment{
        CRDDirectoryPaths:     []string{filepath.Join("..", "...", "config", "crd", "bases")},
        ErrorIfCRDPathMissing: true,
    }

    cfg, err := testEnv.Start()
    Expect(err).NotTo(HaveOccurred())

    // 3. Create scheme and register CRDs
    s := runtime.NewScheme()
    Expect(mycrdv1alpha1.Install(s)).To(Succeed()
    Expect(corev1.AddToScheme(s)).To(Succeed())
    Expect(appsv1.AddToScheme(s)).To(Succeed())
    // Add all external CRD schemas
    Expect(externalcrd.AddToScheme(s)).To(Succeed())

    // 4. Create manager
    mgr, err := ctrl.NewManager(cfg, ctrl.Options{
        Scheme:             s,
        MetricsBindAddress: "0",
        LeaderElection:     false,
    })
    Expect(err).NotTo(HaveOccurred())

    // 5. Add controller to manager
    err = AddToManager(mgr, ...handlerArgs...)
    Expect(err).NotTo(HaveOccurred())

    // 6. Start manager
    go func() {
        defer GinkgoRecover()
        Expect(mgr.Start(ctx)).To(Succeed())
    }()

    // 7. Wait for cache sync
    Eventually(mgr.GetCache().WaitForCacheSync(ctx)).Should(BeTrue())

    k8sClient = mgr.GetClient()
    time.Sleep(2 * time.Second) // Wait for controller to start
})
```

**Important:** All CRD schemas used by the controller and tests must be added to the scheme in BeforeSuite. This includes:
- Your project's CRDs (use `Install(s)` from the API package)
- Core Kubernetes resources (corev1, appsv1, etc.)
- External CRDs (from dependencies like autoscaler, etc.)

## AfterSuite Template

```go
var _ = AfterSuite(func() {
    cancel()
    err := testEnv.Stop()
    Expect(err).NotTo(HaveOccurred())
})
```

## AfterEach Template

```go
var _ = AfterEach(func() {
    // Delete all resources of your CRD type
    list := &mycrdv1alpha1.MyResourceList{}
    Expect(k8sClient.List(context.Background(), list)).Should(Succeed())

    for i := range list.Items {
        By("Deleting: " + list.Items[i].Name)
        Expect(k8sClient.Delete(context.TODO(), &list.Items[i])).Should(Succeed())
        // Wait for deletion
        Eventually(func() bool {
            item := &mycrdv1alpha1.MyResource{}
            err := k8sClient.Get(context.TODO(),
                types.NamespacedName{
                    Name:      list.Items[i].Name,
                    Namespace: list.Items[i].Namespace,
                }, item)
            return errors.IsNotFound(err)
        }, 10*time.Second, 500*time.Millisecond).Should(BeTrue())
    }

    // Delete test namespaces
    nsList := &corev1.NamespaceList{}
    Expect(k8sClient.List(context.Background(), nsList)).Should(Succeed())

    for i := range nsList.Items {
        if strings.HasPrefix(nsList.Items[i].Name, "test-") {
            _ = k8sClient.Delete(context.TODO(), &nsList.Items[i])
        }
    }
})
```

## Testing Patterns

### 1. Basic Resource Creation

```go
var _ = Describe("My Controller", func() {
    Context("When creating a resource", func() {
        It("Should create dependent resources correctly", func() {
            By("creating a test resource")
            resource := &mycrdv1alpha1.MyResource{
                ObjectMeta: metav1.ObjectMeta{
                    Name:      "test-resource",
                    Namespace: "test-ns",
                },
                Spec: mycrdv1alpha1.MyResourceSpec{
                    // ... spec fields
                },
            }
            Expect(k8sClient.Create(ctx, resource)).Should(Succeed())

            By("verifying deployment is created")
            deployment := &appsv1.Deployment{}
            Eventually(func() error {
                return k8sClient.Get(ctx, types.NamespacedName{
                    Name:      "test-resource",
                    Namespace: "test-ns",
                }, deployment)
            }, timeout, interval).Should(Succeed())

            Expect(deployment.Name).To(Equal("test-resource"))
        })
    })
})
```

### 2. State Updates

```go
It("Should update when spec changes", func() {
    resource := createTestResource("test-update", "default")
    Expect(k8sClient.Create(ctx, resource)).Should(Succeed())

    By("updating the resource")
    Eventually(func() error {
        r := &mycrdv1alpha1.MyResource{}
        if err := k8sClient.Get(ctx,
            client.ObjectKeyFromObject(resource), r); err != nil {
            return err
        }
        r.Spec.SomeField = "new-value"
        return k8sClient.Update(ctx, r)
    }, timeout, interval).Should(Succeed())

    By("verifying update reflected in deployment")
    Eventually(func() string {
        d := &appsv1.Deployment{}
        k8sClient.Get(ctx, client.ObjectKey{...}, d)
        return d.Spec.Template.Spec.Containers[0].Image
    }, timeout, interval).Should(Equal("new-image"))
})
```

### 3. Finalizer/Cascade Delete

```go
It("Should handle deletion with finalizers", func() {
    resource := createTestResource("test-delete", "default")
    Expect(k8sClient.Create(ctx, resource)).Should(Succeed())

    By("verifying finalizer is added")
    Eventually(func() bool {
        r := &mycrdv1alpha1.MyResource{}
        k8sClient.Get(ctx, client.ObjectKeyFromObject(resource), r)
        return containsString(r.Finalizers, "my-finalizer")
    }, timeout, interval).Should(BeTrue())

    By("deleting resource")
    Expect(k8sClient.Delete(ctx, resource)).Should(Succeed())

    By("verifying resource is deleted gracefully")
    Eventually(func() bool {
        r := &mycrdv1alpha1.MyResource{}
        err := k8sClient.Get(ctx, client.ObjectKeyFromObject(resource), r)
        return errors.IsNotFound(err)
    }, timeout, interval).Should(BeTrue())
})
```

### 4. Status Synchronization

```go
It("Should sync status from dependent resources", func() {
    resource := createTestResource("test-status", "default")
    Expect(k8sClient.Create(ctx, resource)).Should(Succeed())

    By("waiting for status to be synced")
    Eventually(func() bool {
        r := &mycrdv1alpha1.MyResource{}
        k8sClient.Get(ctx, client.ObjectKeyFromObject(resource), r)
        return r.Status.ReadyReplicas != nil
    }, timeout, interval).Should(BeTrue())

    By("verifying status fields")
    r := &mycrdv1alpha1.MyResource{}
    k8sClient.Get(ctx, client.ObjectKeyFromObject(resource), r)
    Expect(r.Status.Replicas).NotTo(BeNil())
    Expect(r.Status.ReadyReplicas).NotTo(BeNil())
})
```

## Helper Functions

### Resource Creation Helpers

```go
func createTestResource(name, namespace string) *mycrdv1alpha1.MyResource {
    replicas := int32(1)
    return &mycrdv1alpha1.MyResource{
        ObjectMeta: metav1.ObjectMeta{
            Name:      name,
            Namespace: namespace,
        },
        Spec: mycrdv1alpha1.MyResourceSpec{
            Replicas: &replicas,
            // ... other spec fields
        },
    }
}
```

### String Check Helper

```go
func containsString(slice []string, s string) bool {
    for _, item := range slice {
        if item == s {
            return true
        }
    }
    return false
}
```

## Common Issues

### CRD Not Found Errors

**Symptom**: `no matches for kind "SomeCRD" in version "api-group/version"`

**Solution**: Ensure all CRD schemas are added to the scheme in BeforeSuite:

```go
s := runtime.NewScheme()
Expect(mycrdv1alpha1.Install(s)).To(Succeed()
Expect(externalcrd.AddToScheme(s)).To(Succeed()) // Don't forget external CRDs!
```

### External CRD Installation

For external CRDs (like autoscaler.keda.io, keda.sh, etc.), you have two options:

1. **Add to scheme only** (if CRD is already installed in the cluster being tested):
   ```go
   Expect(externalcrd.Install(s)).To(Succeed())
   ```

2. **Also install CRD manifest** (requires CRD YAML file):
   ```go
   testEnv = &envtest.Environment{
       CRDDirectoryPaths: []string{
           filepath.Join("..", "..", "config", "crd", "bases"),
           filepath.Join("..", "..", "vendor", "external-crd-path"), // External CRDs
       },
   }
   ```

### Timing Issues

Use `Eventually` for asynchronous operations:

```go
// ✅ Good: wait for resource creation
Eventually(func() error {
    return k8sClient.Get(ctx, key, resource)
}, timeout, interval).Should(Succeed())

// ✅ Good: wait for status update
Eventually(func() bool {
    r := &mycrdv1alpha1.MyResource{}
    k8sClient.Get(ctx, key, r)
    return r.Status.ReadyReplicas != nil
}, timeout, interval).Should(BeTrue())

// ❌ Bad: no waiting
r := &mycrdv1alpha1.MyResource{}
k8sClient.Get(ctx, key, r)
Expect(r.Status.ReadyReplicas).NotTo(BeNil()) // May fail
```

## Test File Organization

One integration test file per controller:

```
pkg/controllers/
├── mycontroller/
│   ├── mycontroller_controller.go
│   ├── mycontroller_controller_test.go  <-- One file
│   └── ...
├── anothercontroller/
│   ├── anothercontroller_controller.go
│   ├── anothercontroller_controller_test.go  <-- One file
│   └── ...
```

Naming convention: `{crdname}_controller_test.go` where `{crdname}` matches the singular CRD name. If the file already exists and not organized by this skill, try refactor the test file following this skill, otherwise, try add more test cases into it.

## Running Tests

```bash
# Run all tests
go test ./pkg/controllers/...

# Run specific test
go test -v ./pkg/controllers/mycontroller/... -run TestMyController

# Run specific test spec
go test -v ./pkg/controllers/mycontroller/... -ginkgo.focus="When creating a resource"

# Build test binary
go test -c ./pkg/controllers/mycontroller/

# Run with coverage
go test -cover ./pkg/controllers/mycontroller/
```

## Best Practices

1. **Always use `Eventually`** for operations that require reconciliation
2. **Use `Consistently`** for checking negative conditions
3. **Clean up in AfterEach** to prevent test interference
4. **Use descriptive test names** that explain what is being tested
5. **Test both happy path and edge cases**
6. **Verify finalizer handling** for delete operations
7. **Check owner references** for cascade deletion
8. **Mock external handlers** when needed (see example_impl.go pattern)