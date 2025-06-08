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
// controllers/scaletarget_controller.go
```
