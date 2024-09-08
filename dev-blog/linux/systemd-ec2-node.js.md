# systemd를 사용해서 ec2에 node.js 서버 띄우기

보통 node.js를 이용하여 서버를 구축할 때 ec2 인스턴스를 띄우고 pm2로 많이 실행을 하는데요. 저희 회사도 레거시 api는 ec2 인스턴스에서 pm2 cluster모드로 실행이 되고 있습니다.

최근 포스사에서 api 호출이 되지 않는 다는 연락이 왔고, 포스사에서 사용하는 api는 저희 예약관리 서비스를 사용하는 레스토랑에서 포스의 메뉴와 연동할 수 있도록 제공하는 기능이라서 빠르게 조치가 필요했습니다.

여차저차 중지되어있던 서버를 재시작하고, pm2를 재실행해주었습니다.

일부 포스사는 연동이 되었지만, 몇몇 포스사는 계속해서 호출에 실패했습니다.

확인해보니 nginx를 이용해 443포트로 온 요청을 80포트로 프록시 해주고 있었는데, 서버가 다운되면서 nginx가 stop되어 해당 api들은 서버로 전달이 되지 않을 것이었습니다.

이 때문에 재발방지 대책으로 systemd를 사용해 서버가 재시작될때 nginx와 pm2를 모두 자동 실행하도록 설정해주게 되었고, 리서치하게된 systemd로 pm2없이 nodejs 서버를 한번 띄워보고자 합니다.

## 환경 <a href="#headerid_0" id="headerid_0"></a>

환경은 우분투입니다.

## History <a href="#headerid_1" id="headerid_1"></a>

히스토리 명령어는 리눅스 설정이 어떻게 변했는 지 추적할 때 씁니다.

다음과 같이 설정해주면 언제 입력된 명령인지도 확인할 수 있어서 편리합니다.

```bash
vi /etc/profile
### HISTTIMEFORMAT="%Y-%m-%d_%H:%M:%S [CMD]:" 입력
source /etc/profile
```

## 시간 설정 <a href="#headerid_2" id="headerid_2"></a>

```bash
sudo ln -sf /usr/share/zoneinfo/Asia/Seoul /etc/localtime
date
```

위 명령어를 입력해보면 현재 시간이 한국 시간으로 변경된 것을 확인할 수 있습니다.&#x20;

하지만 여기까지만 하면 시간에 대하여 신뢰를 할 수가 없습니다.&#x20;

하드웨어 시간은 실제 시간보다 조금씩 밀리기 때문입니다.

그러다보면 나중에는 몇분에서 몇십분까지 차이가 나는 경우가 있기 때문에 crony라는 것을 사용하여 시간을 계속 동기화해주는 것이 좋습니다.

```bash
sudo apt install chrony
sudo systemctl status chrony
```

active가 나오면 정상적으로 설치 및 동작되고 있는 것입니다.

결과가 다르다면 아래 명령어를 입력해주면 됩니다.

```bash
sudo systemctrl enable chrony
sudo systemctrl start chrony
```

## 기본 패키지 설치 <a href="#headerid_3" id="headerid_3"></a>

```bash
sudo apt update
sudo apt install nodejs
echo "var http = require('http');
var app = http.createServer(function(req, res){
	res.end('hi');
});
app.listen(8080);" > index.js;
```

nodejs를 설치하고 테스트를 위해 요청이오면 hi를 응답하는 index.js 입니다.

```bash
nohup node index.js &
curl http://localhost:8080
```

nohup은 프로세스를 실행한 터미널의 세션 연결이 끊기더라도 프로세스를 계속해서 동작시키는 명령어입니다.&#x20;

로그아웃 등과 같이 세션 연결이 끊기더라도 프로세스가 계속 동작될 수 있도록 합니다. 하지만 Ctrl+C를 누르면 프로세스는 바로 종료되는데요.&#x20;

반면 백그라운드(&) 실행은 실행 시키면 대기 상태가 없지만, 세션 연결이 끊기면 실행한 프로그램도 함께 종료됩니다.&#x20;

해서, 두개를 함께 사용합니다.&#x20;

하지만 이 역시 재부팅시 실행했던 프로세스가 자동으로 재실행되지는 않기에 저희는 systemd가 필요합니다.

## Systemd <a href="#headerid_4" id="headerid_4"></a>

systemd(system daemon)은 전통적으로 Unix 시스템이 부팅후에 가장 먼저 생성된 후에 다른 프로세스를 실행하는 init 역할을 대체하는 데몬입니다.

(deamon은 background에서 실행이 되는 프로세스를 말합니다.)

이전에는 init이라는 데몬이 있었는데 이를 대체하고 init보다 기능이 추가되어서 나온 것이 systemd입니다.

그래서 이전의 init과 같이 PID가 1이 됩니다. 부모프로세스가 없으므로 PPID 또한 1이 됩니다.

먼저 systemd로 index.js를 실행시킬 것이기에 nohup으로 실행시켜둔 프로세스를 종료합니다.

```bash
ps -ef | grep index
<br>
### pid 확인
sudo kill {위에서 확인한 pid}
```

systemd의 아키텍처는 매우 복잡하지만 일반 리눅스 사용자 입장에서는 최상단의 systemd utillities 인 systemctl, journalctl 등 유틸리티 사용법만 익히면 됩니다.

### 서비스 파일 작성

(서비스이름).service 파일을 만들어줍니다. &#x20;

```bash
### index.service
[Unit]
Description=nodejs 테스트

[Service]
ExecStart=node /home/user/index.js
Restart=always
Group=nogroup
Environment=NODE_ENV=development
WorkingDirectory=/home/user/

[Install]
WantedBy=multi-user.target
```

### 시스템 등록

```bash
sudo systemctl link /home/user/index.service
```

### 시스템 자동시작 및 구동

```bash
sudo systemctl enable index
sudo systemctl start index
```

### 상태확인

```bash
sudo systemctl status index
```

active가 나오면 성공입니다.

마지막으로 정말 시스템이 재시작되어도 서버가 동작하는지 확인해보겠습니다.

```bash
sudo reboot
curl http://localhost:8080
```

hi가 나오면 성공!
