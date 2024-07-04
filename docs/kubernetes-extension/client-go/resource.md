## 목차
- [리소스 생성](#리소스-생성)
- [리소스 조회](#리소스-조회)
- [리소스 수정](#리소스-수정)
- [리소스 삭제](#리소스-삭제)

### 리소스 클라이언트 생성(deployment)
```go
    // 1. 리소스 클라이언트 생성
    deploymentsClient := clientset.AppsV1().Deployments(apiv1.NamespaceDefault)
        
    // 2. 리소스 구현체 반환	
    func (c *AppsV1Client) Deployments(namespace string) DeploymentInterface {
        return newDeployments(c, namespace)
    }
    
    func newDeployments(c *AppsV1Client, namespace string) *deployments {
        // 디플로이먼트 구조체를 생성 후 반환
        return &deployments{
            gentype.NewClientWithListAndApply[*v1.Deployment, *v1.DeploymentList, *appsv1.DeploymentApplyConfiguration](
                "deployments",
                c.RESTClient(),
                scheme.ParameterCodec,
                namespace,
                func() *v1.Deployment { return &v1.Deployment{} },
                func() *v1.DeploymentList { return &v1.DeploymentList{} }),
        }
    }
```
- 클라이언트셋 그룹에 명시된 리소스의 클라이언트 생성
- <clientset>.<api-group-version>().<리소스명>(namespace)으로 리소스 구현체 반환


### 리소스 생성
```go
    // 1. 생성할 리소스 선언
	deployment := &appsv1.Deployment{
		ObjectMeta: metav1.ObjectMeta{
			Name: "demo-deployment",
		},
		Spec: appsv1.DeploymentSpec{
			Replicas: int32Ptr(3),
			Selector: &metav1.LabelSelector{
				MatchLabels: map[string]string{
					"app": "demo",
				},
			},
			Template: apiv1.PodTemplateSpec{
				ObjectMeta: metav1.ObjectMeta{
					Labels: map[string]string{
						"app": "demo",
					},
				},
				...
	}
	
	// 2. 리소스 생성
	result, err := deploymentsClient.Create(context.TODO(), deployment, metav1.CreateOptions{})
	
	// 3. create 메소드
	func (c *Client[T]) Create(ctx context.Context, obj T, opts metav1.CreateOptions) (T, error) {
	// 생성된 데이터를 담을 객체 생성
	result := c.newObject()
	err := c.client.Post().
		NamespaceIfScoped(c.namespace, c.namespace != ""). // 네임스페이스 설정
		Resource(c.resource).							   // 리소스 타입 명시
		VersionedParams(&opts, c.parameterCodec).          // 인코딩/디코딩 설정
		Body(obj).										   // 생성할 데이터
		Do(ctx).									       // API 서버에 요청
		Into(result)									   // 결과값을 result 객체에 저장
	return result, err
}
```
- 리소스 클라이언트에 정의된 `Create` 메소드를 사용하여 리소스 생성
- 리소스 생성시 `create(context, 리소스 선언, 생성 옵션)`을 사용하여 리소스 생성
  - 생성 옵션은 다음과 같음
    - DryRun: 리소스 생성 전에 유효성 검사만 수행(실제 생성되지 않고 시뮬레이션만 진행) ex) `DryRun: []string{"All"}
- API 서버에 요청을 보낼 때 method, namespace, 리소스, 인코딩, 데이터를 서버에 보내고 응답을 받음


### 리소스 조회
```go
    // 1. 리로스 클라이언트 생성
    deploymentsClient := clientset.AppsV1().Deployments(apiv1.NamespaceDefault)
    
	// 2. 리소스 조회
    result, getErr := deploymentsClient.Get(context.TODO(), "demo-deployment", metav1.GetOptions{})

	// 3. Get 메소드
    func (c *Client[T]) Get(ctx context.Context, name string, options metav1.GetOptions) (T, error) {
		// 생성된 데이터를 담을 객체 생성
        result := c.newObject()
        err := c.client.Get().                                  
            NamespaceIfScoped(c.namespace, c.namespace != "").   // 네임스페이스 설정
            Resource(c.resource).                                // 조회할 리소스 명시
            Name(name).                                          // 조회할 리소스 네임
            VersionedParams(&options, c.parameterCodec).         // 인코딩/디코딩 설정
            Do(ctx).                                             // API 서버에 요청
            Into(result)                                         // 결과값을 result 객체에 저장
        return result, err
    }
	
	// 4. 리소스 결과는 Deployment(리소스) 구현체 반환
    type Deployment struct {
        metav1.TypeMeta `json:",inline"`
        metav1.ObjectMeta `json:"metadata,omitempty" protobuf:"bytes,1,opt,name=metadata"`
        Spec DeploymentSpec `json:"spec,omitempty" protobuf:"bytes,2,opt,name=spec"`
        Status DeploymentStatus `json:"status,omitempty" protobuf:"bytes,3,opt,name=status"`
}
	// 5. 리소스 리스트 조회
	list, err := deploymentsClient.List(context.TODO(), metav1.ListOptions{})
    
	// 6.리스트 메소드
    func (l *alsoLister[T, L]) List(ctx context.Context, opts metav1.ListOptions) (L, error) {
		...
	    result, err := l.list(ctx, opts)
        if err == nil {
			// watch-cache에 저장된 데이터랑 일관성 검사
			// KUBE_LIST_FROM_CACHE_INCONSISTENCY_DETECTOR 환경 변수가 설정된 경우만 기능 실행
            consistencydetector.CheckListFromCacheDataConsistencyIfRequested(ctx, "list request for "+l.client.resource, l.list, opts, result)
        }
	    return result, err
    }
    
	// 7 리스트 결과는 DeploymentList(리소스 리스트) 구현체 반환
	type DeploymentList struct {
        metav1.TypeMeta `json:",inline"`
        metav1.ListMeta `json:"metadata,omitempty" protobuf:"bytes,1,opt,name=metadata"`
        Items []Deployment `json:"items" protobuf:"bytes,2,rep,name=items"`
    }
	
```
- 리소스 클라이언트에 정의된 `Get, List` 메소드를 사용하여 리소스 조회
- 리소스 조회 시 `get(context, 리소스 이름, 조회 옵션)`을 사용하여 리소스 조회
- 리소스 리스트 조회 시 `list(context, 조회 옵션)`을 사용하여 리소스 리스트 조회
- 단일 리소스 조회 시, 이름을 명시하고 리스트 구현 시 리소스만 명시하여 조회

### 리소스 업데이트
```go
    // 1. 리소스 클라이언트 생성
    deploymentsClient := clientset.AppsV1().Deployments(apiv1.NamespaceDefault)
	
	// 2. 업데이트할 리소스(조회)
    result, getErr := deploymentsClient.Get(context.TODO(), "demo-deployment", metav1.GetOptions{})
	
	// 3. 리소스 업데이트
    result.Spec.Replicas = int32Ptr(2)                           
    result.Spec.Template.Spec.Containers[0].Image = "nginx:1.13"
    result.Annotations = map[string]string{
        "test-v1": "test-v1",
    }
    _, updateErr := deploymentsClient.Update(context.TODO(), result, metav1.UpdateOptions{})
	
	// 4. Update 메소드
    func (c *Client[T]) Update(ctx context.Context, obj T, opts metav1.UpdateOptions) (T, error) {
	// 업데이트된 결과를 담을 오브젝트
	result := c.newObject()
	err := c.client.Put().
		NamespaceIfScoped(c.namespace, c.namespace != ""). // 수정할 리소스 네임스페이스
		Resource(c.resource).							   // 수정할 리소스 명시
		Name(obj.GetName()).							   // 수정할 리소스 이름
		VersionedParams(&opts, c.parameterCodec).          // 인코딩/디코딩
		Body(obj).										   // 수정할 데이터
		Do(ctx).										   // API 서버에 요청
		Into(result)									   // 수정된 결과 저장
	return result, err
}
```
- 리소스 클라이언트에 정의된 `Update` 메소드를 사용하여 리소스 업데이트
- 리소스 객체에서 수정할 값을 설정한 후, `update(context, 리소스, 업데이트 옵션)`을 사용하여 리소스 업데이트