## Proxmox 템플릿 만들기
```shell
# VM ID 9000번으로 템플릿용 VM 생성
qm create 9000 --name "ubuntu-2404-cloudinit-template" \
  --ostype l26 \
  --memory 4096 \
  --cores 4 \
  --cpu host \
  --net0 virtio,bridge=vmbr0
```

```shell
# 가져온 디스크를 SCSI 0번에 할당 (VirtIO SCSI Single 컨트롤러 설정)
qm set 9000 --scsihw virtio-scsi-single \
  --scsi0 local-lvm:vm-9000-disk-0,aio=io_uring,discard=on,iothread=1,ssd=1
```

## K3s 설치
```shell
# 앤서블 파서 및 디스크 관련 컬렉션 설치
ansible-galaxy collection install community.general ansible.posix

# 플레이북 실행 (SSH 패스워드가 필요하다면 -k 추가)
ansible-playbook -i inventory.ini site.yaml
```

## 로컬 kubectl config 설정
```shell
mkdir -p ~/.kube

ssh -i ~/IdeaProjects/terraform-key.pem ubuntu@10.10.60.101 "sudo cat /etc/rancher/k3s/k3s.yaml" > ~/.kube/config

# macOS(BSD) sed는 -i 뒤에 백업 확장자가 필요 (안 쓸 거면 빈 문자열 '')
sed -i '' 's/127.0.0.1/10.10.60.101/g' ~/.kube/config

chmod 600 ~/.kube/config
```

# 재설치 시 수동으로 해야 하는 것 (의존성 순서대로)

ArgoCD가 뭔가를 관리하려면 ArgoCD 자신이 먼저 떠 있어야 하듯, 아래 5개는 전부 "GitOps가 작동하기 위한 전제 조건"이라 순서대로 직접 helm install해야 합니다.

```shell
## 0. 최초 1회 helm repo 등록
helm repo add metallb https://metallb.github.io/metallb
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm repo add traefik https://traefik.github.io/charts
helm repo add jetstack https://charts.jetstack.io
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

## 1. MetalLB
helm install metallb metallb/metallb -n metallb-system --create-namespace
kubectl apply -f metallb/ipaddress-pool.yaml

## 2. Sealed Secrets 컨트롤러
### 1. 백업한 키 먼저 복원
kubectl apply -f sealed-secrets-keys-backup.yaml

### 2. 그 다음 컨트롤러 설치 (최초 부팅 시 위 키를 발견하고 그대로 채택)
helm install sealed-secrets sealed-secrets/sealed-secrets -n kube-system

## 3. ArgoCD
helm install argocd argo/argo-cd -n argocd --create-namespace \
-f apps/argocd/values-argocd.yaml

## 4. 여기서부터 GitOps 시작
kubectl apply -f bootstrap/root-app.yaml
```
