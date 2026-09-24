---
title: 'Xây Dựng Pipeline CI/CD Đa Công Cụ Toàn Diện: Tích Hợp GitLab CI, Jenkins Orchestration, SonarQube Quality Gate Và Ansible Deployment'
date: 2026-09-24 07:35:00 +0700
categories: [DevOps, CICD]
tags: [DevOps, CICD, GitLab, Jenkins, SonarQube, Ansible]
keywords: [DevOps, CICD, GitLab, Jenkins, SonarQube, Ansible]
pin: false
image:
  path: /assets/img/posts/2026/xay-dung-pipeline-ci-cd-da-cong-cu-gitlab-ci-jenkins-sonarqube-ansible/cover.webp
  alt: 'Kiến trúc Pipeline CI/CD đa công cụ phối hợp: GitLab CI kích hoạt Jenkins Declarative Pipeline, SonarQube Quality Gate kiểm định và Ansible tự động hóa triển khai'
---

# I. Dẫn nhập: Lý thuyết & Nguyên lý nền tảng

Chào các bạn! Trong hành trình phát triển phần mềm hiện đại, hầu như tổ chức nào cũng đặt mục tiêu tự động hóa quy trình phân phối sản phẩm (Continuous Integration & Continuous Delivery - CI/CD). Tuy nhiên, khi quy mô doanh nghiệp mở rộng từ một vài nhóm nhỏ lên hàng trăm kỹ sư với hàng nghìn microservices, chúng ta sẽ bắt gặp một nghịch lý quen thuộc: **Không có một công cụ đơn lẻ nào là chiếc "chìa khóa vạn năng" cho tất cả mọi giai đoạn**.

Nhiều giải pháp All-in-One trên thị trường thường xuất sắc ở một vài khâu nhưng lại bộc lộ hạn chế lớn khi bước vào môi trường hạ tầng doanh nghiệp phức tạp (kết hợp on-premise, đa đám mây, các hệ thống bare-metal và máy ảo legacy):
- **GitLab** là nền tảng quản lý mã nguồn (SCM) và cộng tác hàng đầu với tính năng Merge Request approvals, code review và governance phân quyền tuyệt vời.
- **Jenkins** vẫn giữ vững vị thế "ông vua" trong việc điều phối các pipeline phân tán (Distributed Pipeline Orchestration), hỗ trợ ma trận build song song, hàng nghìn agent chuyên dụng (GPU cluster, macOS runner, Windows embedded) và hệ sinh thái plugin đồ sộ.
- **SonarQube** là tiêu chuẩn công nghiệp về Phân tích mã tĩnh (Static Application Security Testing - SAST), theo dõi nợ kỹ thuật (Technical Debt) và thực thi cổng kiểm soát chất lượng (Quality Gate) nghiêm ngặt.
- **Ansible** là công cụ quản lý cấu hình (Configuration Management) và triển khai hạ tầng số một nhờ nguyên lý lũy nghiệm (Idempotency) và kiến trúc Agentless hoạt động mượt mà qua giao thức SSH an toàn.

Thay vì ép buộc một công cụ phải gánh vác những nhiệm vụ mà nó không được thiết kế tối ưu, các kiến trúc sư Platform Engineering hiện đại lựa chọn chiến lược **Toolchain Best-of-Breed**: kết hợp sức mạnh chuyên biệt của từng công cụ thành một pipeline thống nhất, bảo mật và hoàn toàn tự động hóa. Trong bài viết này, mình sẽ cùng các bạn bóc tách kiến trúc tích hợp đa công cụ này, xây dựng kịch bản thực chiến từ khâu commit code đến triển khai rolling update không gián đoạn dịch vụ (Zero-Downtime Deployment).

---

# II. Kiến trúc & So sánh thực tế

### 1. Kiến trúc luồng điều phối Sự kiện (Event-Driven Data Flow)

Toàn bộ quy trình từ khi lập trình viên tạo Merge Request cho tới khi gói phần mềm được cập nhật trên các máy chủ production diễn ra theo luồng dữ liệu sau:

```
[1. Developer Push / MR Merge]
            │
            ▼
┌────────────────────────┐
│   GitLab Repository    │ ──(GitLab CI Job: curl trigger with HMAC token)──┐
└────────────────────────┘                                                  │
                                                                            ▼
                                                           ┌────────────────────────────────┐
                                                           │   Jenkins Master Controller    │
                                                           └────────────────────────────────┘
                                                                    │ (Dispatch Build Job)
                                                                    ▼
                                                           ┌────────────────────────────────┐
                                                           │   Jenkins Kubernetes Agent     │
                                                           │  - Checkout code               │
                                                           │  - Build & Unit Test           │
                                                           │  - Execute SonarQube Scanner   │
                                                           └────────────────────────────────┘
                                                                    │
                                    ┌───────────────────────────────┴───────────────────────────────┐
                                    ▼                                                               ▼
                      ┌───────────────────────────┐                                   ┌───────────────────────────┐
                      │    SonarQube Server       │                                   │   Container Registry      │
                      │  - Calculate Code Smells  │                                   │  - Push Docker Image      │
                      │  - Evaluate Quality Gate  │                                   │    (Nexus / Harbor)       │
                      └───────────────────────────┘                                   └───────────────────────────┘
                                    │
                                    ▼ (Webhook Callback: Quality Gate Status)
                      ┌───────────────────────────┐
                      │ Jenkins: waitForQualityGate│
                      └───────────────────────────┘
                                    │
                  ┌─────────────────┴─────────────────┐
                  │ [Pass Quality Gate]               │ [Fail: Coverage < 80% / Vulnerability]
                  ▼                                   ▼
┌───────────────────────────────────────┐   ┌───────────────────────────────────┐
│ Stage: Ansible Deployment             │   │ Stage: Circuit Breaker Interruption│
│ - Run Ansible Playbook                │   │ - Abort Pipeline                  │
│ - Rolling update target servers       │   │ - Notify Slack / Update MR Status │
│ - Smoke test & Health check           │   └───────────────────────────────────┘
└───────────────────────────────────────┘
```

1. **Giai đoạn Kích hoạt (GitLab CI)**: Khi có commit mới hoặc nhánh `main` được cập nhật, một job nhẹ trong `.gitlab-ci.yml` sử dụng token HMAC để gọi REST API kích hoạt webhook của Jenkins Master.
2. **Giai đoạn Điều phối & Kiểm thử (Jenkins on Kubernetes)**: Jenkins Master lập tức cấp phát một Pod Agent động trên cụm Kubernetes, tiến hành kéo mã nguồn, thực thi build và chạy bộ kiểm thử đơn vị (Unit Tests).
3. **Giai đoạn Gác cổng Chất lượng (SonarQube Quality Gate)**: Jenkins Agent kích hoạt SonarScanner để phân tích mã nguồn và đẩy báo cáo độ bao phủ (Coverage) lên SonarQube. Jenkins bước vào trạng thái chờ webhook phản hồi từ SonarQube qua hàm `waitForQualityGate()`. Nếu Quality Gate báo lỗi (ví dụ: phát hiện lỗ hổng bảo mật cấp Blocker hoặc độ bao phủ kiểm thử dưới 80%), pipeline lập tức ngắt mạch (Circuit Breaker) và dừng triển khai.
4. **Giai đoạn Triển khai Tự động (Ansible)**: Nếu vượt qua Quality Gate, Jenkins gọi Ansible Playbook để thực thi chiến lược cập nhật cuốn chiếu (Rolling Update), cô lập từng node khỏi Load Balancer, nâng cấp container và xác nhận trạng thái sức khỏe qua Health Check trước khi mở lại lưu lượng mạng.

### 2. Bảng phân định trách nhiệm công nghệ trong Pipeline

| Thành phần | Công nghệ | Nhiệm vụ cốt lõi | Input tiếp nhận | Output bàn giao |
| :--- | :--- | :--- | :--- | :--- |
| **SCM & Trigger** | GitLab | Quản lý Git repository, MR approvals, kích hoạt build | Commit / Tag event | Webhook payload kèm Git SHA & Branch |
| **Orchestrator** | Jenkins | Phân phối agent, điều phối pipeline tuần tự/song song | Webhook trigger từ GitLab | Build status, Test Reports, Build Artifacts |
| **Quality Gate** | SonarQube | SAST, phát hiện Security Hotspots, đo lường Code Coverage | Source code & LCOV/JaCoCo coverage report | Quality Gate Result (OK / ERROR) |
| **Deployer** | Ansible | Cấu hình máy chủ, rolling update service, zero-downtime | Artifact image tag, Inventory hosts, Vault keys | Production State verified (HTTP 200) |

---

# III. Cài đặt & Thực hành code: Triển khai thực chiến

### 1. Cấu hình `.gitlab-ci.yml` (GitLab làm Event Emitter)

File cấu hình `.gitlab-ci.yml` này đóng vai trò gửi tín hiệu kèm metadata sang Jenkins khi có thay đổi trên nhánh chính:

```yaml
# .gitlab-ci.yml
stages:
  - notify_orchestrator

trigger_jenkins_pipeline:
  stage: notify_orchestrator
  image: curlimages/curl:8.4.0
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
    - if: '$CI_COMMIT_TAG'
  script:
    - echo "Kích hoạt Jenkins Orchestration Pipeline cho commit ${CI_COMMIT_SHORT_SHA}..."
    - >
      curl -X POST "${JENKINS_URL}/generic-webhook-trigger/invoke" \
        -H "Content-Type: application/json" \
        -H "token: ${JENKINS_WEBHOOK_SECRET_TOKEN}" \
        -d "{
          \"git_repo\": \"${CI_REPOSITORY_URL}\",
          \"git_branch\": \"${CI_COMMIT_REF_NAME}\",
          \"git_commit\": \"${CI_COMMIT_SHA}\",
          \"author\": \"${GITLAB_USER_NAME}\",
          \"commit_msg\": \"${CI_COMMIT_MESSAGE}\"
        }"
```

### 2. Cấu hình `Jenkinsfile` (Declarative Pipeline với Quality Gate & Circuit Breaker)

Pipeline Jenkins dưới đây sử dụng Kubernetes Pod Agent động, tích hợp SonarQube Quality Gate và Ansible deployment:

```groovy
// Jenkinsfile
pipeline {
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: maven-sonar
    image: maven:3.9.5-eclipse-temurin-17
    command: ['cat']
    tty: true
  - name: ansible
    image: cytopia/ansible:latest
    command: ['cat']
    tty: true
'''
        }
    }
    
    environment {
        SONAR_ENV = 'Enterprise-SonarQube'
        REGISTRY = 'registry.internal.corp'
        IMAGE_NAME = 'core-banking/payment-service'
    }

    stages {
        stage('Checkout & Environment Setup') {
            steps {
                echo "Đang xử lý commit: ${env.git_commit} từ branch: ${env.git_branch}"
                checkout scmGit(
                    branches: [[name: "${env.git_commit}"]],
                    userRemoteConfigs: [[url: "${env.git_repo}", credentialsId: 'gitlab-ci-ssh-key']]
                )
            }
        }

        stage('Build & Unit Test') {
            steps {
                container('maven-sonar') {
                    echo "Chạy Unit Tests và xuất báo cáo JaCoCo..."
                    sh 'mvn clean test jacoco:report'
                }
            }
        }

        stage('SonarQube Static Analysis') {
            steps {
                container('maven-sonar') {
                    withSonarQubeEnv(installationName: env.SONAR_ENV) {
                        echo "Chạy SonarScanner đẩy dữ liệu lên SonarQube..."
                        sh '''
                            mvn sonar:sonar \
                              -Dsonar.projectKey=payment-service \
                              -Dsonar.projectName="Payment Microservice" \
                              -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml \
                              -Dsonar.qualitygate.wait=false
                        '''
                    }
                }
            }
        }

        stage('SonarQube Quality Gate Check') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    script {
                        echo "Đang chờ SonarQube webhook trả kết quả Quality Gate..."
                        def qg = waitForQualityGate()
                        if (qg.status != 'OK') {
                            error "DỪNG PIPELINE: SonarQube Quality Gate Thất Bại! Trạng thái: ${qg.status}. Vui lòng sửa lỗi bảo mật hoặc nâng coverage!"
                        }
                        echo "Mã nguồn đạt chuẩn Quality Gate: ${qg.status}"
                    }
                }
            }
        }

        stage('Build & Push Container Image') {
            steps {
                echo "Build immutable container tag: ${env.git_commit}"
                sh "echo 'Pushed image ${REGISTRY}/${IMAGE_NAME}:${env.git_commit}'"
            }
        }

        stage('Ansible Production Deployment') {
            steps {
                container('ansible') {
                    withCredentials([
                        sshUserPrivateKey(credentialsId: 'ansible-prod-ssh-key', keyFileVariable: 'SSH_KEY'),
                        file(credentialsId: 'ansible-vault-secret', variable: 'VAULT_FILE')
                    ]) {
                        echo "Kích hoạt Ansible Playbook triển khai Rolling Update..."
                        sh '''
                            ansible-playbook -i deployment/inventories/production \
                              deployment/deploy.yml \
                              --private-key "${SSH_KEY}" \
                              --vault-password-file "${VAULT_FILE}" \
                              -e "app_version=${git_commit}" \
                              -e "registry_host=${REGISTRY}"
                        '''
                    }
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        failure {
            echo "Pipeline thất bại! Bắn thông báo cảnh báo qua Telegram/Slack..."
        }
        success {
            echo "Triển khai production thành công mỹ mãn!"
        }
    }
}
```

### 3. Cấu hình Ansible Playbook với Rolling Update Zero-Downtime

File `deployment/deploy.yml` thể hiện kỹ thuật cập nhật từng máy chủ (serial: 1), ngắt tải tại Load Balancer, nâng cấp container và kiểm tra tính sẵn sàng trước khi đưa trở lại cụm:

```yaml
---
# deployment/deploy.yml
- name: Zero-Downtime Rolling Update Payment Service
  hosts: payment_servers
  serial: 1                     # Cập nhật từng node một (Rolling update)
  max_fail_percentage: 0        # Nếu 1 node fail thì dừng toàn bộ playbook ngay lập tức
  become: true

  vars:
    app_port: 8080
    health_check_url: "http://127.0.0.1:{{ app_port }}/actuator/health"

  tasks:
    - name: 1. Tạm dừng lưu lượng mạng tại Load Balancer (Drain connection)
      ansible.builtin.command:
        cmd: "/usr/local/bin/lb-cli drain --host {{ inventory_hostname }}"
      delegate_to: localhost
      changed_when: false

    - name: 2. Kéo Container Image mới nhất từ Private Registry
      community.docker.docker_image:
        name: "{{ registry_host }}/core-banking/payment-service:{{ app_version }}"
        source: pull
        force_source: true

    - name: 3. Khởi động Container mới với Non-root Security
      community.docker.docker_container:
        name: payment-service-app
        image: "{{ registry_host }}/core-banking/payment-service:{{ app_version }}"
        state: started
        restart_policy: always
        user: "10001:10001"
        read_only: true
        ports:
          - "{{ app_port }}:8080"
        env:
          DB_PASSWORD: "{{ vault_db_password }}"
        recreate: true

    - name: 4. Kiểm tra Health Check Endpoint (Chờ ứng dụng sẵn sàng)
      ansible.builtin.uri:
        url: "{{ health_check_url }}"
        status_code: 200
        return_content: yes
      register: health_result
      until: "health_result.status == 200 and 'UP' in health_result.content"
      retries: 12
      delay: 5

    - name: 5. Kích hoạt lại lưu lượng mạng tại Load Balancer
      ansible.builtin.command:
        cmd: "/usr/local/bin/lb-cli un-drain --host {{ inventory_hostname }}"
      delegate_to: localhost
      changed_when: false
```

---

# IV. Lesson learned & Tổng kết: Best Practices & Cạm bẫy production

Từ kinh nghiệm triển khai thực tế trên các hệ thống tài chính và thương mại điện tử lớn, mình muốn lưu ý với các bạn 4 điểm then chốt:

1. **Bẫy treo vô hạn tại `waitForQualityGate()`**:
   - Khi cấu hình Jenkins lắng nghe kết quả từ SonarQube, nếu firewall chặn chiều gọi ngược từ SonarQube về Jenkins hoặc địa chỉ Server Base URL trong SonarQube bị sai, webhook callback sẽ bị mất. Khi đó Jenkins sẽ treo mãi mãi! Luôn bọc bước này trong `timeout(time: 5, unit: 'MINUTES')`.
2. **Tuyệt đối không dùng tag `:latest` trong môi trường sản xuất**:
   - Việc deploy container với tag `:latest` là một thảm họa khi cần rollback. Hãy luôn gắn nhãn image bằng Git commit hash duy nhất (`${GIT_COMMIT}`).
3. **Bảo mật bí mật với Ansible Vault và Centralized Secrets Management**:
   - Tuyệt đối không commit password database hay SSH key vào repo Git. Hãy mã hóa bằng Ansible Vault hoặc tích hợp HashiCorp Vault để truyền credentials trực tiếp vào RAM trong thời gian chạy playbook.
4. **Áp dụng chính sách "Clean as You Code" trong SonarQube**:
   - Đối với các dự án lớn, việc đặt ngưỡng 100% test coverage cho toàn bộ codebase sẽ khiến đội ngũ phát triển nản lòng và tê liệt tiến độ release. Thay vào đó, hãy thiết lập Quality Gate chỉ kiểm định mã mới (New Code): yêu cầu 80%+ coverage và 0 blocker bug trên các dòng code vừa viết.

Hy vọng kiến trúc đa công cụ này sẽ giúp các bạn có cái nhìn tổng thể và xây dựng được những đường ống dẫn mã nguồn an toàn, ổn định và tự động hóa cao nhất cho tổ chức của mình!
