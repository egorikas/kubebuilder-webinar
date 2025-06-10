# kubebuilder-webinar
My preparation notes for the webinar I am going to host


## Setup of the env.
### Step 1.
Check, that docker runs `docker --version`

### Step 2.
Install kubectl `brew install kubernetes-cli`

### Step 3.
Install kubebuilder `brew install kubebuilder`

### Step 4.
Install [k3d](https://k3d.io/stable/) `brew install k3d`

### Step 5.
Create a new cluster `k3d cluster create kubebuilder-webinar`

### Step 6.
Check the env:
* `kubectl cluster-info`
* `kubectl get nodes`
* `kubectl config get-contexts | grep "kubebuilder"`

## CRD.
* `mkdir ~/scaleoperator`
* `kubebuilder init --domain webinar.io --repo webinar.io/scaleoperator`
* `kubebuilder create api --group scale --version v1alpha1 --kind Scaler`
* Add code
```
/*
Copyright 2025.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
*/

package v1alpha1

import (
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// ScalerSpec defines the desired state of Scaler.
type ScalerSpec struct {
	// Desired replicas from infrastructure actor
	InfraReplicas *int32 `json:"infraReplicas,omitempty"`
	// Desired replicas from application portal actor
	PortalReplicas *int32 `json:"portalReplicas,omitempty"`
	// Target Deployment name to scale
	DeploymentName string `json:"deploymentName"`
}

// ScalerStatus defines the observed state of Scaler.
type ScalerStatus struct {
	// ObservedGeneration is the most recent generation observed for this Scaler.
	ObservedGeneration int64 `json:"observedGeneration,omitempty"`
	// DesiredReplicas is the number of replicas that the Scaler is trying to achieve.
	DesiredReplicas int32 `json:"desiredReplicas"`
	// ActualReplicas is the number of replicas that are currently running.
	ActualReplicas int32 `json:"actualReplicas"`
	// Conditions represent the latest available observations of a Scaler's current state.
	Conditions []metav1.Condition `json:"conditions,omitempty"`
}

// Scaler is the Schema for the scalers API.
// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
// +kubebuilder:printcolumn:name="Desired",type="integer",JSONPath=".status.desiredReplicas"
type Scaler struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Spec   ScalerSpec   `json:"spec,omitempty"`
	Status ScalerStatus `json:"status,omitempty"`
}

// ScalerList contains a list of Scaler.
// +kubebuilder:object:root=true
type ScalerList struct {
	metav1.TypeMeta `json:",inline"`
	metav1.ListMeta `json:"metadata,omitempty"`
	Items           []Scaler `json:"items"`
}

func init() {
	SchemeBuilder.Register(&Scaler{}, &ScalerList{})
}
```

## Delivery.
* `make`
* `make install`
* `kubectl get crd`
* `kubectl get crd scalers.scale.webinar.io -o yaml`

## Controller.
Controller code:
```
package controller

import (
	"context"
	"fmt"
	appsv1 "k8s.io/api/apps/v1"
	corev1 "k8s.io/api/core/v1"
	apierrors "k8s.io/apimachinery/pkg/api/errors"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/runtime"
	"k8s.io/apimachinery/pkg/types"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/controller/controllerutil"
	"sigs.k8s.io/controller-runtime/pkg/log"
	scalev1alpha1 "webinar.io/scaleoperator/api/v1alpha1"
)

// ScalerReconciler reconciles a Scaler object
type ScalerReconciler struct {
	client.Client
	Scheme *runtime.Scheme
}

// Reconcile is part of the main kubernetes reconciliation loop which aims to
// +kubebuilder:rbac:groups=scale.webinar.io,resources=scalers,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=scale.webinar.io,resources=scalers/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=scale.webinar.io,resources=scalers/finalizers,verbs=update
func (r *ScalerReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	logger := log.FromContext(ctx)
	var scaler scalev1alpha1.Scaler
	if err := r.Get(ctx, req.NamespacedName, &scaler); err != nil {
		if apierrors.IsNotFound(err) {
			return ctrl.Result{}, nil
		}
		return ctrl.Result{}, err
	}
	// if Scale is being deleted, we can handle cleanup logic here
	if scaler.DeletionTimestamp != nil {
		// Handle deletion logic if necessary
		return ctrl.Result{}, nil
	}

	deployment, err := r.getDeploymentForScale(ctx, &scaler)
	if err != nil {
		logger.Error(err, "Failed to get or create Deployment for scaling", "deployment", scaler.Spec.DeploymentName)
		return ctrl.Result{}, err
	}

	desiredReplicas := getDesiredReplicas(&scaler)
	if desiredReplicas == nil {
		logger.Info("No replicas specified for scaling", "scale", scaler.Name)
		return ctrl.Result{}, nil
	}

	current := int32(0)
	if deployment.Spec.Replicas != nil {
		current = *deployment.Spec.Replicas
	}
	// Check if the desired replicas are already set
	// Prevent unnecessary updates and loops
	if *desiredReplicas == current {
		return ctrl.Result{}, nil
	}

	deployment.Spec.Replicas = desiredReplicas
	if err := r.Update(ctx, deployment); err != nil {
		return ctrl.Result{}, err
	}

	err = r.updateStatus(ctx, &scaler, *desiredReplicas, current)
	if err != nil {
		logger.Error(err, "Failed to update Scaler status", "scaler", scaler.Name)
		return ctrl.Result{}, err
	}

	logger.Info("Scaler reconciled successfully", "scaler", scaler.Name, "desiredReplicas", *desiredReplicas, "actualReplicas", current)
	return ctrl.Result{}, nil
}

func getDesiredReplicas(scaler *scalev1alpha1.Scaler) *int32 {
	if scaler.Spec.InfraReplicas != nil {
		return scaler.Spec.InfraReplicas
	}
	return scaler.Spec.PortalReplicas
}

func (r *ScalerReconciler) getDeploymentForScale(ctx context.Context, scaler *scalev1alpha1.Scaler) (*appsv1.Deployment, error) {
	var deploy appsv1.Deployment
	err := r.Get(ctx, types.NamespacedName{Name: scaler.Spec.DeploymentName, Namespace: scaler.Namespace}, &deploy)
	if err != nil && !apierrors.IsNotFound(err) {
		log.FromContext(ctx).Error(err, "failed to get Deployment for Scale", "deployment", scaler.Spec.DeploymentName)
		return nil, err
	}
	if apierrors.IsNotFound(err) {
		deploy = *newDeploymentForScale(scaler)
		if err := controllerutil.SetControllerReference(scaler, &deploy, r.Scheme); err != nil {
			return nil, err
		}
		err = r.Create(ctx, &deploy)
		if err != nil {
			log.FromContext(ctx).Error(err, "failed to create Deployment for Scale", "deployment", deploy.Name)
			return nil, err
		}
		log.FromContext(ctx).Info("Deployment created", "deployment", deploy.Name)
	}
	return &deploy, nil
}

func newDeploymentForScale(scale *scalev1alpha1.Scaler) *appsv1.Deployment {
	replicas := int32(1)
	if scale.Spec.InfraReplicas != nil {
		replicas = *scale.Spec.InfraReplicas
	} else if scale.Spec.PortalReplicas != nil {
		replicas = *scale.Spec.PortalReplicas
	}
	labels := map[string]string{
		"app": scale.Name,
	}
	return &appsv1.Deployment{
		ObjectMeta: metav1.ObjectMeta{
			Name:      scale.Spec.DeploymentName,
			Namespace: scale.Namespace,
			Labels:    labels,
		},
		Spec: appsv1.DeploymentSpec{
			Replicas: &replicas,
			Selector: &metav1.LabelSelector{
				MatchLabels: labels,
			},
			Template: corev1.PodTemplateSpec{
				ObjectMeta: metav1.ObjectMeta{Labels: labels},
				Spec: corev1.PodSpec{
					Containers: []corev1.Container{{
						Name:  "app",
						Image: "nginx:stable",
						Ports: []corev1.ContainerPort{{ContainerPort: 80}},
					}},
				},
			},
		},
	}
}

func (r *ScalerReconciler) updateStatus(ctx context.Context, scaler *scalev1alpha1.Scaler, desiredReplicas int32, actualReplicas int32) error {
	scaler.Status.DesiredReplicas = desiredReplicas
	scaler.Status.ActualReplicas = actualReplicas
	scaler.Status.ObservedGeneration = scaler.GetGeneration()
	// Update the status conditions
	condition := metav1.Condition{
		Type:               "Scaled",
		Status:             metav1.ConditionTrue,
		Reason:             "ScalingSuccessful",
		Message:            fmt.Sprintf("Scaler is successfully scaling the deployment to %d replicas", desiredReplicas),
		ObservedGeneration: scaler.GetGeneration(),
	}
	// there is a problem here
	scaler.Status.Conditions = append(scaler.Status.Conditions, condition)

	return r.Status().Update(ctx, scaler)
}

// SetupWithManager sets up the controller with the Manager.
func (r *ScalerReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return ctrl.NewControllerManagedBy(mgr).
		For(&scalev1alpha1.Scaler{}).
		Named("scaler").
		Complete(r)
}

```

* `k3d image import nginx:stable -c kubebuilder-webinar`
* Create the object `kubectl apply -f scale_v1alpha1_scaler.yaml`
