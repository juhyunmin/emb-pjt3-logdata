# [project3] 로그데이터 관리 (Node.js) new

# 3주차 프로젝트 :  ****로그데이터 관리 Application 제작

## 프로젝트 요구사항 및 과정 설계

개발을 하다보면 필수적으로 로그데이터를 종합해서 조회해야하는 일들이 생겨난다. 

로그 데이터를 웹 소켓으로 전달받아 실시간으로 로그를 조회하는 서비스를 만들어 보자.

A.	만들어야 하는 기능

1. 기초 서버 세팅 및 웹 소켓 연결하기 
2. 로그 라이브러리를 이용해 로그 기록하기
3. 웹 소켓을 활용해서 로그 데이터 전송하기

먼저 프로젝트 디렉터리는 다음 두 개의 하위 디렉터리로 나뉜다.

`./server` : Express Server

`./client` : `index.html` 파일

두 개의 디렉터리를 루트로 설정해, 두 개의 VSCode 를 켜준다.

`./server` 디렉터리에 `index.js` 를 생성하고, `Ctrl + `` 을 사용해 터미널을 열어 `npm init` 을 입력한다.

필요한 패키지를 다운로드받자.

```bash
$ npm i express cors socket.io winston winston-daily-rotate-file
```

## **기초 서버 세팅 및 socket.io 연결하기**

index.js

```jsx
const express = require("express");
const PORT = 8080;
const { createServer } = require("http");
const { Server } = require("socket.io");

const app = express();

app.use(express.json());

// 서버 객체 생성
const httpServer = createServer(app);

// cors 설정
const cors = require("cors");
app.use(cors());
//io에 대한 cors 설정은 따로 해줘야 한다.
const io = new Server(httpServer, {
  cors: true,
});

// 연결되었을 때
io.on("connection", (socket) => {
  console.log("연결 되었음");
});

app.get("/bug", (req, res) => {
  return res.json({
    test: "Hello",
  });
});

httpServer.listen(PORT, () => console.log(`${PORT} 서버 기동중`));

```

웹소켓 라이브러리인 socket.io를 사용하기 위해서는 http 서버가 필요하다. 
따라서 express 기반 http 서버를 만들어주어 사용한다. 

작성하였다면 nodemon index를 통해 서버를 실행시켜준다. 

서버쪽의 소켓을 생성하였으니 

이제 해당 소켓과 연결해주는 클라이언트 부분을 만들어주면 된다. 

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Socket.io Example</title>
  </head>
  <body>
    <h1>Socket.io Example</h1>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/socket.io/4.7.2/socket.io.js"></script>
    <script>
      const socket = io("http://localhost:8080");

      socket.on("connect", () => {
        console.log("서버와 연결되었습니다.");
      });

      socket.on("disconnect", () => {
        console.log("서버와의 연결이 끊어졌습니다.");
      });
    </script>
  </body>
</html>

```

클라이언트는 라이브서버로 작동하지 않고, `index.html` 을 더블클릭해 열어서 확인하도록 하겠다. REST 요청 시 새로고침이 발생하는 버그가 발생하기 때문이다.

정상적으로 동작하는 경우  서버에서 `연결 되었음` 이 출력되게 된다. 

클라이언트를 확인하면, 맨 처음 `서버와 연결되었습니다.` 구문이 출력되지만, 서버와 연결 종료 시 다음 결과가 나온다.

![Untitled](public/Untitled.png)

즉, socket.io 를 사용하면 클라이언트는 서버와 연결을 주기적으로 체크해, 데이터를 실시간으로 가져올 수 있다.

언제 쓰이는가?

- 채팅 사이트
- 멀티플레이어 게임
- 주기적으로 갱신되는 차트

정말인지 확인하기 위해, 1초마다 클라이언트에 chicken 이라는 데이터를 보내는 코드를 작성해보자.

`server/index.js`

```jsx
// 연결되었을 때
io.on("connection", (socket) => {
  console.log("연결 되었음");

  setInterval(() => {
    socket.emit("fromServer", "chicken");
  }, 1000);
});
```

`client/index.html`

```jsx
const socket = io("http://localhost:8080");

socket.on("connect", () => {
  console.log("서버와 연결되었습니다.");
});

// 추가된 부분
socket.on("fromServer", (data) => {
  console.log(data);
});

socket.on("disconnect", () => {
  console.log("서버와의 연결이 끊어졌습니다.");
});
```

위 내용 대로 작성하고 테스트해보면

![Untitled](public/Untitled%201.png)

1초마다 클라이언트에 chicken이라는 데이터가 잘 전송되게 된다.

## 로그 라이브러리를 활용해 로그 기록하기

로그 데이터 프로젝트는 지금까지 배운 Node.js 를 사용하여, 서버를 구성하고 운영하기 위한 로그 프로그램을 만들어 서버의 로그를 실시간으로 확인하는 데 목적이 있다. 수업시간에 한 내용에 이어, 로그를 시각적으로 Visualization 한다.

## 통신 API 작성

`winston`  은 로그 기록을 위한 패키지다. 로그란, 서버에서 일어나는 일을 기록한 일기장이며, 일기장의 주요도는 레벨로 구분된다.

주로 다섯가지 레벨에 따라, 다음 상황에서 로그는 기록된다.

DEBUG : 개발 시 버그 발생. 배포시에는 꺼줘야 한다.

INFO : 일반적인 상황

WARN : 경고

ERROR : 에러

FATAL : 시스템에 치명적인 에러!

서버 관리자는 서버에 기록된 로그를 보고 현재 서버의 상태를 판단한다.

그런데 왜 레벨이라고 표현했을까? 다음 상황을 가정하자.

- 개발이 끝난지는 꽤 되었고, 서버도 잘 작동하는것 같으니, 별 것 아닌 경고까진 무시해야겠다.

이 경우, 개발자가 로그 레벨을 ERROR 로 지정하면, DEBUG, INFO, WARN 은 로그가 발생하지 않는다. 즉, 상황에 따라 "레벨" 을 지정할 수 있기 때문에 로그 레벨이라고 표현하는 것이다.

또한, 만들고자하는 서버 프로그램에 따라 개발자는 특정 상황에서의 로그 레벨을 직접 지정할 수 있다.

- 만약 주문 수량이 다 떨어졌는데 주문이 계속되면 아주 큰일나는 상황이니, 해당 상황 발생 시 ERROR 로그를 출력해야겠다.

시스템 입장에선 주문 수량이 다 떨어진 게 별 것 아닐 수 있다. 시스템에 영향을 끼치는 일이 아니기 때문이다. 그러나 회사엔 아주 큰 영향을 끼칠 수 있으므로, 적절한 상황 발생 시 개발자는 해당 상황에 따른 로그 레벨을 지정할 수 있는 것이다.

이런 로그를 관리하는 JavaScript 패키지가 바로 winston 이다.

설정파일을 만들어야한다.

`server/` 디렉터리 하위에, `utils/` 디렉터리를 생성하고, `winston.js` 파일 생성

```jsx
const winston = require("winston");
require("winston-daily-rotate-file");

const transport = new winston.transports.DailyRotateFile({
  level: "info",
  filename: "./logs/%DATE%.log",
  datePattern: "YYYY-MM-DD-HH",
  zippedArchive: true,
  maxSize: "20m",
  maxFiles: "1d",
  format: winston.format.combine(
    winston.format.timestamp({ format: "YYYY-MM-DD HH:mm:ss" }),
    winston.format.printf((log) => {
      return `${log.timestamp} [${log.level}]: ${log.message}`;
    })
  ),
});

const logger = winston.createLogger({
  transports: [transport],
});

module.exports = logger;
```

`level` : 현재 `info` 로 지정했으므로, 로그 레벨 원칙에 따르면 `debug` 는 출력되지 않는다.

`filename` : 로그 파일명이다. 날짜 기준으로 저장하겠다.

`datePattern` : 날짜 시간 규칙

`zippedArchive` : 충분히 쌓이면 zip 으로 압축할것인지의 여부

`maxSize` : 최대 크기. 20메가바이트 초과 시, 새로운 로그 파일 생성

`maxFiles` : 로그 파일의 최대 갯수. 1d 이므로, 하루동안 생성된 로그만 저장하며, 갯수 초과 시 가장 오래된 로그파일부터 삭제

`format` : 로그가 저장될 형태

이제 index.js 로 옮겨가서, 해당 설정파일을 가져오고 일부러 ERROR 로그를 일으켜보자.

```jsx
const logger = require("./utils/winston");

// ...

app.get("/bug", (req, res) => {
  // 추가
  logger.error("에러가 발생하였습니다.");
  return res.json({
    test: "Hello",
  });
});

httpServer.listen(PORT, () => console.log(`${PORT} 서버 기동중`));
```

자, 발생시키고 싶다면? `GET /bug` 요청을 보내면 된다.

![Untitled](public/Untitled%202.png)

요청을 보내게 되면 

![Untitled](public/Untitled%203.png)

날짜.log 파일 형식으로 로그가 저장된것을 확인할 수 있다. 

## 웹 소켓을 활용해서 로그 데이터 전송하기

이제, 저장된 로그를 1초마다 클라이언트로 전송해보자.

현재 로그는 파일로 저장되어있기 때문에, 파일 입출력을 사용해야한다.

`server/index.js`

```jsx
const fs = require('fs');

// ...

function sendLog(socket) {
  //디렉터리를 읽는 함수
  fs.readdir("./logs", (error, files) => {
    if (error) {
      console.error(`ERROR ${error}`);
    } else {
      // .log로 끝나는 파일만 모으기
      const logFiles = files.filter((file) => file.endsWith(".log"));

      //log파일중 첫번째 읽기
      fs.readFile(`./logs/${logFiles[0]}`, (err, data) => {
        const log = data.toString("utf-8");
        socket.emit("fromServer", log);
      });
    }
  });
}

// 연결되었을 때
io.on("connection", (socket) => {
  console.log("연결 되었음");

  setInterval(() => {
    // socket.emit("fromServer", "chicken");

    // 1초마다 로그 감지
    sendLog(socket);
  }, 1000);
});
```

readdir - 디렉터리 읽기

readfile - 파일 읽기 

logs 폴더 디렉토리를 읽어서 그에 해당하는 첫번째 파일을 읽어서 

나타낸다.

![Untitled](public/Untitled%204.png)

## 심화과제

1. 로그 정보를 시각화 하기
2. error 메시지 말고도 서버 코드를 수정해서 다양한 log가 출력 되도록 하기
3. error 메시지만 조회할수 있도록 해보기

**예시** 

![Untitled](public/Untitled%205.png)