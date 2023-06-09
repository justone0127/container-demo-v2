## Account Day - Container Workshop

컨테이너 기술의 특징을 확인하고, 가상머신과의 차이점을 알아보고, 동일한 애플리케이션을 컨테이너 플랫폼에 배포하여 확인해보는 핸즈온 워크샵입니다.

```
목차
1. Workshop 환경 설명
2. Apache 웹 서버 (HTTPD) 설치
3. 어플리케이션 배포 
4. 웹서버 버전 업그레이드 
5. 웹서버 버전 롤백 
```

<br/>

### 1. 환경 설명

Workshop 환경은 OpenShift Cluster 환경에서 실행되는 OpenShift Virtualization(VM) 환경으로 각 사용자마다 RHEL 환경이 제공됩니다.

- **OpenShift**
  - `OpenShift Virtualization` : OpenShift 환경에서 실행되는 가상화 환경
  - `Web Terminal` : OpenShift 환경에서 실행되는 VM접속 및 CLI를 실행 할 수 있는 Web Terminal 환경
  - 각 사용자마다 Lab을 실행할 수 있는 계정과 프로젝트가 제공됩니다. 
    - 프로젝트 : `userx-vm` 이름으로 구성 되어 있으며, OpenShift 콘솔에 자신이 부여 받은 계정으로 접속하면 해당 프로젝트를 확인 할 수 있습니다.
    - VM 계정 정보 : <span style="color: red">userx (x는 숫자입니다) </span> / 비밀번호 :   <span style="color: red">openshift</span>

**1-1) VM 접속 방법**

- OpenShift Console 접속 : 제공 받은 Console URL 정보를 가지고 웹 브라우저를 통해 접속 합니다. 

  - 계정/비밀번호 : <span style="color: red"> userx / openshift </span>

  ![console_connect](https://github.com/justone0127/container-demo-v2/blob/main/openshift_images/console_connect.png)

- `userx-vm` 프로젝트를 선택합니다.

  ![user_project_select](https://github.com/justone0127/container-demo-v2/blob/main/openshift_images/user_project_select.png)

- 콘솔 상단의 **(>_)** 아이콘을 선택하여 Web Terminal에 접속합니다.

  ![web_terminal_icon](https://github.com/justone0127/container-demo-v2/blob/main/openshift_images/web_terminal_icon.png)

- 아래 OpenShift command line terminal에서 `userx-vm` 프로젝트가 선택 되었는지 확인 후, Start 버튼을 누릅니다. 

  ![web_terminal_start](https://github.com/justone0127/container-demo-v2/blob/main/openshift_images/web_terminal_start.png)

- 터미널을 새 창으로 열기 위해서는 새 창으로 오픈 아이콘을 선택하면 새로운 웹 브라우저에서 터미널이 실행됩니다.

  ![new_terminal](https://github.com/justone0127/container-demo-v2/blob/main/openshift_images/new_terminal.png)

- SSH 서비스 확인

  ```bash
  oc get svc
  ```

  ![vm_svc](https://github.com/justone0127/container-demo-v2/blob/main/openshift_images/vm_svc.png)

- VM 접속

  ```bash
  ssh userx@$CLUSTER-IP -p 22000
  ```

  ![vm_ssh_connect](https://github.com/justone0127/container-demo-v2/blob/main/openshift_images/vm_ssh_connect.png)

- `sudo` 권한 스위치

  패키지 설치 및 Labs을 실행을 위해 현재 계정에서 `sudo` 권한으로 스위치합니다.

  ```bash
  sudo -i
  ```

  ![13_sudo_switch](https://github.com/justone0127/container-demo-v2/blob/main/openshift_images/13_sudo_switch.png)

### 2. Apache 웹 서버 (HTTPD) 설치

---

Apache 웹 서버를 RHEL OS와 컨테이너에 각각 구성해보면서 각각의 특징 및 차이점을 확인합니다.

먼저, Web Terminal을 통해 자신의 VM에 접속하여 `sudo` 권한으로 스위치 되었는지 확인한 후 Lab을 진행합니다.



**2-1) 가상머신 기반 리눅스에 httpd 설치**

Red Hat Enterprise Linux 8 운영체제에서 패키지 관리자 도구인 dnf를 통해 설치 가능한 httpd 버전을 확인합니다.

```bash
dnf list --showduplicate httpd
```

![14_httpd_install_version_check](https://github.com/justone0127/container-demo-v2/blob/main/openshift_images/14_httpd_install_version_check.png)

Red Hat Enterprise Linux 8 운영체제에서 패키지 관리자 도구인 dnf를 통해 httpd 서비스를 설치합니다.

```bash
dnf install -y httpd-2.4.37-47.module+el8.6.0+14529+083145da.1.x86_64
```

Apache 웹 서버인 httpd 데몬의 서비스 포트를 8080으로 변경합니다.

```bash
sed -i 's/Listen 80/Listen 8080/g' /etc/httpd/conf/httpd.conf
```

Apache 웹 서버인 httpd 서비스를 활성화하고 기동합니다. 

```bash
systemctl enable httpd
systemctl start httpd
```

페이지 호출은 OpenShift의 관리자 콘솔에서 `userx-vm` 프로젝트에서 **Networking** 선택 > **Route** 선택 > **http-8080**의 Location (주소)를 통해 확인합니다.

![15_http_8080](https://github.com/justone0127/container-demo-v2/blob/main/openshift_images/15_http_8080_route.png)

다음과 같이 Apache HTTP Server의 기본 index.html 페이지가 확인됩니다.

![16_yum_httpd_8080](https://github.com/justone0127/container-demo-v2/blob/main/openshift_images/16_yum_httpd_8080.png)

**2-2) httpd 컨테이너를 사용하여 기동**

Red Hat에서 제공하는 검증된 httpd 이미지를 다운로드 받습니다. 
podman pull 명령어로 다운로드를 진행합니다.

만약, 다음과 같이 인증 에러가 발생하는 경우에는 홈 디렉토리 위치에 있는 `07_podman_login.sh` 쉘 스크립트를 수행한 후 이미지 다운로드를 진행합니다.

- 인증 에러 출력 메시지

  ![17_podman_login_authentication](https://github.com/justone0127/container-demo-v2/blob/main/openshift_images/17_podman_login_authentication.png)

- podman login 쉘 실행

  ```bash
  ./07_podman_login.sh
  ```

  ![18_podman_login](https://github.com/justone0127/container-demo-v2/blob/main/openshift_images/18_podman_login.png)

Red Hat에서 제공하는 검증된 httpd 이미지를 다운로드 받습니다. 
podman pull 명령어로 다운로드를 진행합니다.

```bash
podman pull registry.redhat.io/rhel8/httpd-24:1-166
```

![19_imags_pull](https://github.com/justone0127/container-demo-v2/blob/main/openshift_images/19_imags_pull.png)

다운로드 받은 이미지를 확인합니다.

```bash
podman images
```

![20_podman_images](https://github.com/justone0127/container-demo-v2/blob/main/openshift_images/20_podman_images.png)

다운로드 받은 httpd 이미지를 실행하여 웹 서버 서비스를 확인합니다.

```bash
podman run -d --name httpd -p 8081:8080 registry.redhat.io/rhel8/httpd-24:1-166
```

![21_podman_httpd_256_run](https://github.com/justone0127/container-demo-v2/blob/main/openshift_images/21_podman_httpd_166_run.png)

실행한 컨테이너의 프로세스를 확인합니다.

```bash
podman ps
```

![22_podman_httpd_256_process](https://github.com/justone0127/container-demo-v2/blob/main/openshift_images/22_podman_httpd_166_process.png)

페이지 호출은 OpenShift의 관리자 콘솔에서 `userx-vm` 프로젝트에서 **Networking** 선택 > **Route** 선택 > **http-8081**의 Location (주소)를 통해 확인합니다.

![23_http_8081_route](https://github.com/justone0127/container-demo-v2/blob/main/openshift_images/23_http_8081_route.png)

다음과 같이 Apache HTTP Server의 기본 index.html 페이지가 확인됩니다.

![24_podman_container_httpd_8081](https://github.com/justone0127/container-demo-v2/blob/main/openshift_images/24_podman_container_httpd_8081.png)

**2-3) 요약 - 웹 서버 설치시 특징**

일반적인 가상머신의 OS 환경과 컨테이너 환경에서 웹 서버 구성의 시간 차이는 별로 나지 않았습니다.  

<br/>

### 3. 애플리케이션 배포
---

간단한 웹 페이지 기반의 게임 애플리케이션을 웹서버에 배포하여 서비스를 확인합니다. 
Git 소스에 있는 개발 소스를 로컬에 다운로드 합니다.

```bash
git clone https://github.com/ellisonleao/clumsy-bird/
ls ./clumsy-bird/
```

**3-1) 가상머신 기반 리눅스의 httpd 웹 서버에 APP 배포**

다운로드 받은 게임 소스를 웹 서버가 바라보는 애플리케이션 디렉터리 위치로 복사합니다.

```bash
# 애플리케이션 소스 위치 확인
cat /etc/httpd/conf/httpd.conf | grep DocumentRoot

# 기존 소스 백업
tar cvf app_old.tar /var/www/html

# 애플리케이션 소스 복사 
ls ./clumsy-bird/
cp -R ./clumsy-bird/* /var/www/html/
ls /var/www/html
```

*Apache 웹 서버의 DocumentRoot 설정 확인*

```bash
cat /etc/httpd/conf/httpd.conf | grep DocumentRoot
```

![25_httpd_documentroot](https://github.com/justone0127/container-demo-v2/blob/main/openshift_images/25_httpd_documentroot.png)

*애플리케이션 복사*

```bash
ls ./clumsy-bird/
cp -R ./clumsy-bird/ /var/www/html/
ls /var/www/html/
```

![26_application_copy](https://github.com/justone0127/container-demo-v2/blob/main/openshift_images/26_application_copy.png)

애플리케이션 복사가 완료되었으면, 웹 브라우저에서 게임 서비스를 확인합니다.

![15_http_8080_route](https://github.com/justone0127/container-demo-v2/blob/main/openshift_images/15_http_8080_route.png)

다음과 같이 게임 서비스로 호출 됨을 확인합니다.

![27_vm_game_app](https://github.com/justone0127/container-demo-v2/blob/main/openshift_images/27_vm_game_app.png)



**3-2) httpd 컨테이너에 APP 배포**

기존에 서비스하던 httpd 컨테이너를 중지합니다. 

```bash
podman stop httpd
```

registry.redhat.io/rhel8/httpd-24:1-256 컨테이너에 개발 소스를 배포하는 Containerfile을 생성합니다.

```bash
cat <<EOF > Containerfile
FROM registry.redhat.io/rhel8/httpd-24:1-166

# Add application sources
RUN rm -rf /var/www/html/*
ADD ./clumsy-bird/ /var/www/html/

# The run script uses standard ways to run the application
CMD run-httpd
EOF
```

Containerfile 명세 파일을 활용하여 컨테이너 이미지를 생성합니다.

```bash
podman build -t httpd-game:1-166 .
```

![28_podman_build](https://github.com/justone0127/container-demo-v2/blob/main/openshift_images/28_podman_build.png)

새로 빌드된 이미지를 확인합니다.

```bash
podman images
```

![29_podman_game_images](https://github.com/justone0127/container-demo-v2/blob/main/openshift_images/29_podman_game_images.png)

새롭게 만든 httpd-game 이미지를 활용하여 컨테이너를 기동합니다.

```bash
podman run -d --name httpd-game-1-166 -p 8081:8080 httpd-game:1-166
```

![30_podman_game_166_run](https://github.com/justone0127/container-demo-v2/blob/main/openshift_images/30_podman_game_166_run.png)

실행된 Container의 프로세스를 확인합니다.

```bash
podman ps
```

![31_podman_game_166_process](https://github.com/justone0127/container-demo-v2/blob/main/openshift_images/31_podman_game_166_process.png)

웹 브라우저에서 게임 서비스를 확인합니다.

![23_http_8081_route](https://github.com/justone0127/container-demo-v2/blob/main/openshift_images/23_http_8081_route.png)

다음과 같이 게임 서비스로 호출 됨을 확인합니다.

![32_podman_game_app](https://github.com/justone0127/container-demo-v2/blob/main/openshift_images/32_podman_game_app.png)



**3-3) 요약 - 웹 어플리케이션 배포시 특징**

<span style="color: blue">VM 기반 리눅스의 웹 서버에 새로운 애플리케이션을 배포하는 경우에는 기존 소스의 백업이 필요하지만, 웹 서버 컨테이너를 활용하는 경우에는 podman 명령어 실행시 소스 위치만 잡아주면 되므로 더 간단합니다.</span>

<br/>


### 4. 웹서버 버전 업그레이드
---

**4-1) 리눅스에 설치된 httpd 웹 서버 업그레이드**

httpd 버전을 확인합니다.  
현재 버전은 <span style="color: green">httpd-2.4.37-47.module+el8.6.0+14529+083145da.1</span> 입니다.

```bash
dnf list --showduplicate httpd
```

![33_rhel8_httpd_version_before](https://github.com/justone0127/container-demo-v2/blob/main/openshift_images/33_rhel8_httpd_version_before.png)

더 최신 버전인 <span style="color: green">httpd-2.4.37-56.module+el8.8.0+18758+b3a9c8da.6.x86_64</span> 으로 httpd 웹 서버를 업그레이드합니다.

```bash
dnf update -y httpd-2.4.37-56.module+el8.8.0+18758+b3a9c8da.6.x86_64
```

httpd 버전이 <span style="color: red">2.4.37-56.module+el8.8.0+18758+b3a9c8da.6</span> 으로 업그레이드된 것을 확인합니다.

```bash
dnf list --showduplicate httpd
```

![34_rhel8_httpd_version_after](https://github.com/justone0127/container-demo-v2/blob/main/openshift_images/34_rhel8_httpd_version_after.png)

웹 브라우저에서 http-8080 서비스의 정상 유무를 확인합니다.

![15_http_8080_route](https://github.com/justone0127/container-demo-v2/blob/main/openshift_images/15_http_8080_route.png)

**4-2) httpd 컨테이너의 웹 서버 업그레이드**

Podman 명령어로 httpd 컨테이너 이미지의 최근 Tag를 확인합니다.

```bash
podman search --list-tags registry.redhat.io/rhel8/httpd-24
```

![35_podman_search_tags](https://github.com/justone0127/container-demo-v2/blob/main/openshift_images/35_podman_search_tags.png)

기본 베이스가 되는 httpd 컨테이너의 버전 (<span style="color: red">1-256</span>)을 확정하고 Containerfile을 수정합니다.


```bash
cat Containerfile
```

![36_containerfile](https://github.com/justone0127/container-demo-v2/blob/main/openshift_images/36_containerfile.png)

파일 내용에 컨테이너 버전을 수정합니다.

```bash
sed -i 's/1-166/1-256/g' ./Containerfile
```

파일에서 버전이 제대로 수정이 되었는지 확인합니다.

```bash
cat Containerfile
```

![37_podman_tag_update](https://github.com/justone0127/container-demo-v2/blob/main/openshift_images/37_podman_tag_update.png)

Containerfile 명세 파일을 활용하여 컨테이너 이미지를 생성합니다.

```bash
podman build -t httpd-game:1-256 .
```

![38_podman_image_build_256](https://github.com/justone0127/container-demo-v2/blob/main/openshift_images/38_podman_image_build_256.png)

새로 <span style="color: red">1-256</span>버전으로 빌드된 컨테이너 이미지를 확인합니다.

```bash
podman images
```

![39_podman_game_image_256](https://github.com/justone0127/container-demo-v2/blob/main/openshift_images/39_podman_game_image_256.png)

기존 실행 중인 httpd-game-1-166 컨테이너는 중지합니다.


```bash
podman stop httpd-game-1-166
```

![40_podman_166_stop](https://github.com/justone0127/container-demo-v2/blob/main/openshift_images/40_podman_166_stop.png)

httpd-game-1-166 컨테이너가 중지됐는지 확인합니다.

```bash
podman ps
```

![41_podman_166_stop_ps](https://github.com/justone0127/container-demo-v2/blob/main/openshift_images/41_podman_166_stop_ps.png)

 새로운 버전의 httpd-game:1-256 이미지를 활용하여 컨테이너를 기동합니다.

```bash
podman run -d --name httpd-game-1-256 -p 8081:8080 httpd-game:1-256
```

![42_podman_256_start](https://github.com/justone0127/container-demo-v2/blob/main/openshift_images/42_podman_256_start.png)

httpd-game-1-256 컨테이너가 실행중인 프로세스를 확인합니다.

![43_podman_256_process](https://github.com/justone0127/container-demo-v2/blob/main/openshift_images/43_podman_256_process.png)

웹 브라우저에서 http-8081 서비스의 정상 유무를 확인합니다.

![23_http_8081_route](https://github.com/justone0127/container-demo-v2/blob/main/openshift_images/23_http_8081_route.png)

**4-3) 요약 - 웹 서버 업그레이드**

실습에서는 VM환경과 컨테이너 환경에서의 웹 서버 버전 업그레이드에 대해 확인해 보았습니다. VM의 경우에는 리눅스 환경에 설치된 패키지 버전을 확인하여 신규 버전으로 업그레이드 하였지만, 실제 업무 환경에서는 현재 사용 중인 OS와 설치할 웹 서버 혹은 WAS 서버의 버전 호환성 등을 체크하여 신규로 구성하는 것이 보편적입니다. 반면, 컨테이너 환경의 경우 실습에서 진행했던 것처럼 실제 환경에서도 OS 버전에 대한 호환성을 체크하지 않고도 기본 베이스가 되는 컨테이너의 태그(버전)만 수정하여 새로 빌드하여 실행이 가능합니다. 

<br/>

### 5. 웹서버 버전 롤백
---

**4-1) 리눅스에 설치된 httpd 웹 서버 버전 롤백**

기존의 httpd 웹 서버 버전인 <span style="color: green">2.4.37-47.module+el8.6.0+14529+083145da.1</span> 로 다시 롤백합니다.

```bash
dnf list --showduplicate httpd
dnf downgrade -y httpd-2.4.37-47.module+el8.6.0+14529+083145da.1.x86_64
```
버전이 롤백된 것을 확인합니다.

```bash
dnf list --showduplicate httpd
```
![44_rhel8_httpd_version_rollback](https://github.com/justone0127/container-demo-v2/blob/main/openshift_images/44_rhel8_httpd_version_rollback.png)

**5-2) httpd 컨테이너의 웹 서버 버전 롤백**

신규 버전의 httpd-game-1-256 프로세스를 확인합니다.

```bash
podman ps
```

![45_podman_256_process_01](https://github.com/justone0127/container-demo-v2/blob/main/openshift_images/45_podman_256_process_01.png)

신규 버전의 httpd-game-1-256 컨테이이너를 중지합니다.

```bash
podman stop httpd-game-1-256
```

![46_podman_256_stop_02](https://github.com/justone0127/container-demo-v2/blob/main/openshift_images/46_podman_256_stop_02.png)

이전 버전의 httpd-1-166 컨테이너를 실행합니다.

```bash
podman start httpd-game-1-166
```

![47_podman_166_start_03](https://github.com/justone0127/container-demo-v2/blob/main/openshift_images/47_podman_166_start_03.png)

이전 버전의 httpd-1-166 컨테이너가 실행됐음을 확인합니다.

![48_podman_166_process_04](https://github.com/justone0127/container-demo-v2/blob/main/openshift_images/48_podman_166_process_04.png)

<br/>
