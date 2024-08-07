# 외부 API 주입 스크립트 문제

<img width="1439" alt="image" src="https://github.com/user-attachments/assets/698f1b90-bf7b-468f-8165-17e3c5435413">

``` shell
Cannot read properties of undefined (reading 'Places')
TypeError: Cannot read properties of undefined (reading 'Places')
   at handleSearch (http://localhost:3000/main.6915dc271f6c9793d622.hot-update.js:71:44)
    at HTMLUnknownElement.callCallback (http://localhost:3000/static/js/bundle.js:16167:18)
    at Object.invokeGuardedCallbackDev (http://localhost:3000/static/js/bundle.js:16211:20)
    at invokeGuardedCallback (http://localhost:3000/static/js/bundle.js:16268:35)
    at invokeGuardedCallbackAndCatchFirstError (http://localhost:3000/static/js/bundle.js:16282:29)
    at executeDispatch (http://localhost:3000/static/js/bundle.js:20425:7)
    at processDispatchQueueItemsInOrder (http://localhost:3000/static/js/bundle.js:20451:11)
    at processDispatchQueue (http://localhost:3000/static/js/bundle.js:20462:9)
    at dispatchEventsForPlugins (http://localhost:3000/static/js/bundle.js:20471:7)
    at http://localhost:3000/static/js/bundle.js:20631:16
```

# 문제 상황
카카오 맵 API를 사용하는데 Places 모듈을 못가져오는 문제
카카오맵 API를 가져오는 것과 별개로 Places 모듈을 사용하려면 다른 URL을 추가로 가져와줘야하는데 컴파일 에러가 발생하지 않고 해당 모듈을 가져오는 것만 문제가 생기는 것으로 봐서 이 부분에서 문제가 생긴 것이 아닐지 추측

# 해결 방안
index.html에 선언되어있는 AppKey가 현재 2개 설정되어있는데 스크립트 파일을 한 개로 만들어줘야 함

## 문제 코드
```
<script type="text/javascript" src="//dapi.kakao.com/v2/maps/sdk.js?appkey={app-key}"></script>
<script type="text/javascript" src="//dapi.kakao.com/v2/maps/sdk.js?appkey={app-key}&libraries=services"></script>
```

## 해결 코드
```
<script type="text/javascript" src="//dapi.kakao.com/v2/maps/sdk.js?appkey={app-key}&libraries=services"></script>
```
