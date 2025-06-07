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

## Operator creation.
* `mkdir ~/scaleoperator`
* `kubebuilder init --domain webinar.io --repo webinar.io/scaleoperator`
* `kubebuilder create api --group scale --version v1alpha1 --kind Scaler`
* Add code
```
// ScalerSpec defines the desired state of Scaler.
type ScalerSpec struct {
	// TargetDeploymentName is the name of the Kubernetes Deployment
	// this ScaleOperator should manage.
	// +kubebuilder:validation:Required
	// +kubebuilder:validation:MinLength=1
	TargetDeploymentName string `json:"targetDeploymentName"`

	// InfrastructureReplicas is the desired number of replicas
	// as specified by the infrastructure actor.
	// This field has higher priority.
	// +kubebuilder:validation:Minimum=0
	// +optional
	InfrastructureReplicas *int32 `json:"infrastructureReplicas,omitempty"`

	// ApplicationPortalReplicas is the desired number of replicas
	// as specified by the application portal actor.
	// This field has lower priority.
	// +kubebuilder:validation:Minimum=0
	// +optional
	ApplicationPortalReplicas *int32 `json:"applicationPortalReplicas,omitempty"`
}

// ScalerStatus defines the observed state of Scaler.
type ScalerStatus struct {
	// ObservedGeneration reflects the.metadata.generation of the ScaleOperator CR
	// that was last reconciled by the controller.
	// +optional
	ObservedGeneration int64 `json:"observedGeneration,omitempty"`

	// CalculatedDesiredReplicas is the number of replicas the operator has decided
	// should be set on the target Deployment after applying priority logic.
	// +optional
	CalculatedDesiredReplicas *int32 `json:"calculatedDesiredReplicas,omitempty"`

	// ActualReplicas is the current number of ready replicas of the managed Deployment.
	// Corresponds to Deployment.status.readyReplicas.
	// +optional
	ActualReplicas *int32 `json:"actualReplicas,omitempty"`

	// TargetDeploymentObservedSpecReplicas is the.spec.replicas field of the target Deployment
	// as last observed or set by this operator.
	// +optional
	TargetDeploymentObservedSpecReplicas *int32 `json:"targetDeploymentObservedSpecReplicas,omitempty"`

	// Conditions store the status conditions of the ScaleOperator resource.
	// +operator-sdk:csv:customresourcedefinitions:type=status
	// Conditions metav1.Condition `json:"conditions,omitempty" patchStrategy:"merge" patchMergeKey:"type" protobuf:"bytes,1,rep,name=conditions"`
}


// +kubebuilder:object:root=true
// +kubebuilder:subresource:status

// Scaler is the Schema for the scalers API.
type Scaler struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Spec   ScalerSpec   `json:"spec,omitempty"`
	Status ScalerStatus `json:"status,omitempty"`
}

// +kubebuilder:object:root=true

// ScalerList contains a list of Scaler.
type ScalerList struct {
	metav1.TypeMeta `json:",inline"`
	metav1.ListMeta `json:"metadata,omitempty"`
	Items           []Scaler `json:"items"`
}

``` 
