##### 본 실습은 kubeadm 으로 k8s 1.29 가 구성되어 있는 환경에서 실습 진행함 #####
1) run time : cri-dockerd
2) master : 172.16.8.10, worker1,2,3 : 172.16.8.11, 172.16.8.12, 172.16.8.13 (NAT)
3) metallb 구축되어 있음 > iprange : 172.16.8.201 ~ 172.16.8.239
4) master 노드에 nfs 서버 구축되어 있으며 /k8s 디렉토리를 공유 디렉토리로 사용(777)
- master : apt install -y nfs-server nfs-common
- worker : apt install -y nfs-common


##### 시작할 때 'kubectl create namespace daa-stack' 로 네임 스페이스 생성 #####


##### 01-zookeeper.yaml #####
kubectl apply -f 01-zookeeper.yaml > ZooKeeper 서비스 및 StatefulSet 생성
kubectl get pods -n daa-stack -w > 3개 파드가 모두 1/1 Running이 되는지 확인
kubectl logs daa-zoo-0 -n daa-stack > 로그 끝에 QuorumPeer가 보이면 정상



##### 02-kafka.yaml #####
kubectl apply -f 02-kafka.yaml > Kafka 서비스 및 StatefulSet 생성
kubectl logs daa-kafka-0 -n daa-stack > KafkaServer id=0 started 메시지 확인
kubectl exec -it daa-kafka-0 -n daa-stack -- kafka-topics.sh --create --topic my-topic --bootstrap-server localhost:9092 > 테스트용 토픽 생성



##### 03-elk.yaml #####
kubectl apply -f 03-elk.yaml > "ES, Kibana 배포 및 외부 서비스 노출"
kubectl get svc -n daa-stack > "elasticsearch-lb, kibana-svc의 EXTERNAL-IP 확인"
curl http://[ES-EXTERNAL-IP]:9200 



##### 04-logstash.yaml #####
kubectl apply -f 04-logstash.yaml > Logstash 설정 및 배포
kubectl logs -f deployment/logstash -n daa-stack > Pipeline started 및 Successfully joined group 확인



##### 전체 동작 상태 검증 #####
kubectl exec -it daa-kafka-0 -n daa-stack -- \
kafka-console-producer.sh --topic my-topic --bootstrap-server localhost:9092
메시지 입력 예: {"msg": "hello cloud"}

브라우저에서 http://[KIBANA-IP]:5601 접속.
Index Pattern 생성: kafka-app-log-*
Discover 메뉴: 실시간으로 들어오는 로그 확인.



재시작이 필요할 때: kubectl rollout restart statefulset [이름] -n daa-stack
데이터 초기화: kubectl delete pvc --all -n daa-stack (주의: NFS의 실제 데이터도 삭제될 수 있음)
