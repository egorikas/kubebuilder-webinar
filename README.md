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
// ScalerSpec defines the desired state of Scaler.
type ScalerSpec struct {
	// Desired replicas from infrastructure actor
	InfrastructureReplicas int32 `json:"infrastructureReplicas,omitempty"`
	// Desired replicas from application portal actor
	ApplicationReplicas int32 `json:"applicationReplicas,omitempty"`
	// Target Deployment name to scale
	DeploymentName string `json:"deploymentName,omitempty"`
}

// ScalerStatus defines the observed state of Scaler.
type ScalerStatus struct {
	AppliedReplicas int32 `json:"appliedReplicas,omitempty"`
	ActualReplicas  int32 `json:"actualReplicas,omitempty"`
}

// Scaler is the Schema for the scalers API.
// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
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
```

## Delivery.
* `make`
* `make install`
* `kubectl get crd`
* `kubectl get crd scalers.scale.webinar.io -o yaml`

## Controller.
