---
layout: post
title: "Electron에서 alert 사용시 input 읽기 전용 문제 해결"
date: 2024-05-13
categories: [Electron, troubleshooting]
description: "Electron 애플리케이션에서 alert 함수 사용시 발생하는 input 필드의 읽기 전용 문제 해결 방법을 알아봅니다."
---

## 개요

Electron 애플리케이션에서 `alert()` 함수를 사용할 때, input 태그가 읽기 전용으로 설정되는 현상이 종종 발생합니다. 이 현상은 Electron의 구조적 특성 때문에 나타나는 문제로, 이해를 돕기 위해 문제의 원인과 해결 방안을 상세히 설명하겠습니다.

## 원인 분석

Electron은 Chromium과 Node.js를 기반으로 하는데, 이는 Electron이 브라우저와 유사한 환경을 제공하지만, 일부 특성이 다르다는 것을 의미합니다. 일반적인 웹 브라우저에서는 `alert()`, `prompt()`, `confirm()` 같은 함수들이 동기적으로 실행되어, 사용자의 입력을 기다리는 동안 전체 렌더링 엔진을 멈춥니다. 그러나 Electron에서는 이러한 함수들이 메인 프로세스의 시스템 대화 상자를 통해 실행되며, 이 과정에서 렌더러 프로세스는 계속해서 비동기적으로 동작합니다.

이 때문에, 메인 스레드가 시스템 모달을 실행하면서 렌더러 프로세스의 입력 필드 등이 일시적으로 읽기 전용 상태가 될 수 있습니다. 이는 메인 프로세스와 렌더러 프로세스가 독립적으로 작동하면서 발생하는 동기화 문제로 볼 수 있습니다.

## 해결 방안

### 비동기 대화 상자 사용하기

Electron의 `dialog` 모듈을 사용하여 비동기적으로 메시지 박스를 표시할 수 있습니다. 이 방법은 렌더러 프로세스를 차단하지 않고 사용자의 입력을 받을 수 있습니다.

```javascript
const { dialog } = require("electron");

dialog
  .showMessageBox({
    type: "info",
    title: "Information",
    message: "This is an important message."
  })
  .then((result) => {
    console.log(result.response);
  });
```

### 사용자 정의 모달 구현하기

HTML과 CSS를 활용하여 사용자 정의 모달을 만들어 사용하는 것도 좋은 방법입니다. 이는 UI/UX를 완전히 제어할 수 있으며, 기존의 웹 기술을 활용하므로 개발자에게 친숙합니다.

```
<div id="myModal" class="modal">
  <div class="modal-content">
    <span class="close">&times;</span>
    <p>Some text in the Modal..</p>
  </div>
</div>
```

```
<style>
.modal {
    display: none;
    position: fixed;
    z-index: 1;
    left: 0;
    top: 0;
    width: 100%;
    height: 100%;
    overflow: auto;
    background-color: rgb(0,0,0);
    background-color: rgba(0,0,0,0.4);
}
</style>
```

```

const modal = document.getElementById("myModal");
const span = document.getElementsByClassName("close")[0];

span.onclick = function() {
    modal.style.display = "none";
}

window.onclick = function(event) {
    if (event.target == modal) {
        modal.style.display = "none";
    }
}


```

## 결론

Electron에서 alert() 함수의 사용은 입력 필드를 일시적으로 읽기 전용으로 만들 수 있습니다. 이 문제를 해결하기 위해 비동기 대화 상자 사용이나 사용자 정의 모달 구현을 추천합니다. 이 방법들을 통해 사용자 경험을 개선하고 애플리케이션의 효율성을 높일 수 있습니다.
커스텀 alert나 modal, dialog 사용함으로써 Electron의 에러를 회피해야합니다. 처음 이 에러를 접하면 어떠한 동작때문인가 싶을텐데
해당 이벤트(클릭,handler함수 실행의 사이드이페트)보단 alert그 자체의 문제였어서 당황하였습니다.
