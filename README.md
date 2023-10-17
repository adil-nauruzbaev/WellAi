# WellAI
![Logo](https://github.com/GaLenN3228/wellai_mobile/blob/master/assets/logo.png?raw=true)

## About app

WellAI is an app for patients and doctors from Bent Tree Family. This is the one of the best clinics in Frisco and North Dallas. The app contains a lot of features which are shown in the "Features" section. We split doctor and patient roles into two apps because all functions are specified for one specific role. It is quite hard and confusing having all roles in one app. Due to the previous reason, app for the doctor contains additional nurse role. Although doctor can be a patient with the same email.

## 🛠 App languages
Mobile app: Dart and flutter<br />
Backend: Go<br />
Web app: Vue.js

## Features

- Chat with nurse or doctor  [example](https://github.com/GaLenN3228/wellai_mobile/tree/master/examples/chat)
- Virtual assistant to help with diagnoses   [example](https://github.com/GaLenN3228/wellai_mobile/tree/master/examples/assistant)
- WebRTS calling to nurse or doctor  [example](https://github.com/GaLenN3228/wellai_mobile/tree/master/examples/calls)
- Authorization with google or apple [example](https://github.com/GaLenN3228/wellai_mobile/tree/master/examples/google_and_apple_auth)
- Attach to doctor or nurse in calendar [example](https://github.com/GaLenN3228/wellai_mobile/tree/master/examples/calendar)
- Profile editing [example](https://github.com/GaLenN3228/wellai_mobile/tree/master/examples/profile_editing)

## Chat with nurse or doctor

Chat with nurse or doctor implementation 

For chat system we use Web socket for connection and handling messages. For WebSocket connection we use [socket_io_client](https://pub.dev/packages/socket_io_client). The connection example provided here:

```dart
    _socket = socket_io.io(
        baseUrl,
        <String, dynamic>{
          'transports': ['websocket'],
          'reconnection': true,
          'timeout': 20000,
          'forceNew': true,
          'auth': {'token': 'Bearer ${_tokensRepository.accessToken}'},
        },
      );
      _socket.onConnectError((data) {
        log('connect error', name: _tag);
      });
      _socket.on('error', (data) async {
        log(data.toString(), name: _tag);
      });
      _socket.onDisconnect((data) {
        log('disconnect', name: _tag);
      });
      _socket.onReconnect((data) {
        log('reconnect', name: _tag);
        _initWebRTC();
      });
      _socket.onConnect((data) {
        networkInfo.updateConnection(true);
        log('onConnect', name: _tag);
        _initWebRTC();
      });
```

Main event from socket handler: 

```dart
     void _initWebRTC() {
    _onEvent(WsCancelResponse.name);
    _onEvent(WsCallResponse.name);
    _onEvent(WsMediaOfferResponse.name);
    _onEvent(WsMediaAnswerResponse.name);
    _onEvent(WsApproveResponse.name);

    _onEvent(WsChatSendedMessageResponse.name);
    _onEvent(WsJoinChatResponse.name);
    _onEvent(WSLeaveChatResponse.name);
    _onEvent(WsRemotePeerIceCandidateResponse.name, checkFunction: (data) {
      return data['candidate'] != null;
    });
    _onEvent(WsChatMessageResponse.name);
    for (var event in WSScreenUpdateResponse.subValues) {
      _onEvent(event);
      emit(event, SimpleEncodable({}));
    }
  }
```

and _onEvent fucntion: 

```dart
    _onEvent(
        String event, {
        bool Function(dynamic)? checkFunction,
        VoidCallback? customCallBack,
    }) {
        _socket.off(event);
        _socket.on(event, (data) async {
        log('SocketEvent: ' + event);
         log('Event data: ${jsonEncode(data)}');
        if (checkFunction?.call(data) ?? true) {
            _eventStreamController.sink.add(WsResponse.fromMap(event, data));
            customCallBack?.call();
        }
    });
  }
```

Due to chat bloc complex structure we only provide implementation, in this example you can find connection to chat, and listener method, which listens events from previous part. All completed examples can be found in "Features" section:

```dart
  NurseChatBloc(
    this._chatWrapper,
    this._viewModel,
    this._repository,
    this._userStore,
    this._networkInfo,
  ) : super(LoadingNurseChatState()) {
    on<SendMessageEvent>(_onSendMessageEvent);
    on<GetMessageEvent>(_onGetMessageEvent);
    on<InitialNurseChatEvent>(_onInitialNurseChatEvent);
    on<ReconnectNurseChatEvent>(_onReconnectNurseChatEvent);
    _messagesSub = _chatWrapper.messagesStream.where((event) {
      final ev = event as WsChatMessageResponse;
      return ev.chatId == _chatId;
    }).listen((event) {
      add(GetMessageEvent(event as WsChatMessageResponse));
    });
    _networkConnectionSub = _networkInfo.reconnectionStream.listen((event) {
      if (event == true) {
        add(ReconnectNurseChatEvent());
      }
    });
  }
```

<p align="center">
  <img src="https://github.com/GaLenN3228/wellai_mobile/blob/master/assets/chat.gif" alt="animated" width="300" height="600"/>
</p>

## Virtual assistant to help you with diagnoses

Virtual assistant is an AI which was programmed by software engineers from the USA and provided for us. This AI is helping us to give the user more common diagnoses depending on a few questions. All questions from assistant are playing from phone speakers and user can answer from phone microphone.

Code logic for assistant is pretty same as for chat.


```dart
    class AssistantBloc extends Bloc<AssistantEvent, AssistantState> {
    AssistantBloc(
        this._repository,
        this._lang,
        this._assistantViewModel,
        this._userStore,
        this._chatMessageFinalizer,
        this._audioPlayer,
        this._settingsManager,
        this._screenUpdateManager,
        this._sessionId,
  ) : super(InitialDataState()) {
    on<SuccessFillProfileAssistantEvent>(_onSuccessFillProfileAssistantEvent);
    on<SendMessageEvent>(_onSendMessageEvent);
    on<StartOverAssisEvent>(_onStartOverAssisEvent);
    on<StopSessionAssisEvent>(_onStopSessionAssisEvent);
  }
```

All work from SendMessageEvent event, when we get response from server: 

```dart
    void _onSendMessageEvent(SendMessageEvent event, Emitter<AssistantState> emit) async {
    if (event.message.trim().isEmpty) return;
    _assistantViewModel.changeAssistantAnimatedState(true);

    final currentAssistStage = assistantStage;
    _assistantViewModel.changeAssistWidgetsState(true);

    if (currentAssistStage is AssisStageInit) {
      await _handleInitStage(event, currentAssistStage, emit);
    }

    if (currentAssistStage is AssisStageClarificationOfSymptom) {
      _handleClarificationOfSymptomStage(event, currentAssistStage, emit);
    }

    if (currentAssistStage is AssisStageAskSymptoms) {
      await _handleAskSymptomsStage(event, currentAssistStage, emit);
    }

    if (currentAssistStage is AssisStageWaitingForRetry) {
      await _handleWaitingForRetry(currentAssistStage, emit, event);
    }
  }
```

<p align="center">
  <img src="https://github.com/GaLenN3228/wellai_mobile/blob/master/assets/assistant.gif" alt="animated" width="300" height="600"/>
</p>


## WebRTS calling to nurse or doctor

Calling in app has the same solution like we use in chat. We connect to Web socket, Web socket provides us events which we will handle in BLoC. All events and their handlings are available. [example](https://github.com/GaLenN3228/wellai_mobile/tree/master/examples/calls)

```dart 
_wsSubscription = _stream.listen((event) {
      if (event is WsCancelResponse) {
        _onCancelEvent(event);
        return;
      }
      if (event is WsCallResponse) {
        _onCallEvent(event);
        return;
      }
      if (event is WsMediaOfferResponse) {
        _onMediaOfferEvent(event);
        return;
      }
      if (event is WsMediaAnswerResponse) {
        _onMediaAnswerResponse(event);
        return;
      }
      if (event is WsRemotePeerIceCandidateResponse) {
        _onRemotePeerIceCandidateEvent(event);
        return;
      }
      if (event is WsApproveResponse) {
        _onApproveEvent(event);
      }
    });
```

Call event handling 

```dart 
 Future<void> _onCallEvent(WsCallResponse event) async {
    room = event.room;
    var constraints = {
      "audio": true,
      "video": {'facingMode': 'user', 'optional': []}
    };
    try {
      myStream = await navigator.mediaDevices.getUserMedia(constraints);
      _callStreamController.sink.add(
          CallModel(CallStoreState.incomingCall, callerName: event.callerName));
    } catch (e) {
      _callStreamController.sink
          .add(CallModel(CallStoreState.permissionNotGranted));
      final miceStatus = await Permission.microphone.status;
      final cameraStatus = await Permission.camera.status;
      if (miceStatus.isDenied || cameraStatus.isDenied) {
        _sendPermissionDenied(event.room);
      }
    }
  }
```


## Authorization with google or apple

Authorization with google or apple works with firebase. All implementations are quite simple. We just create wrappers for google and apple.

Google auth wrapper:

```dart
class GoogleSignInWrapper {
  final List<String> scopes;

  GoogleSignInWrapper({required this.scopes});

  Future<String?> getGoogleIdToken() async {
    final googleAccount = await GoogleSignIn(scopes: scopes).signIn();
    final auth = await googleAccount?.authentication;
    return auth?.idToken;
  }

  Future<void> logoutWithGoogle() async {
    await GoogleSignIn().disconnect();
  }
}
```

Apple wrapper: 

```dart
class SignInWithAppleWrapper {
  final List<AppleIDAuthorizationScopes> scopes;

  SignInWithAppleWrapper({required this.scopes});

  Future<AuthorizationCredentialAppleID>? getAppleIDCredential() async {
    return SignInWithApple.getAppleIDCredential(scopes: scopes);
  }
}
```

For each platform we create events in BLoC: 

```dart 
  void _onSignInWithGoogleEvent(
      SignInWithGoogleEvent event, Emitter<SignInState> emit) async {
    try {
      final googleAccount = await _googleSignIn.getGoogleIdToken();
      if (googleAccount != null) {
        final token = await _globalRepository.loginWithGoogle(googleAccount,
            isLogin: event.isLogin);
        await _googleSignIn.logoutWithGoogle();
        emit(SuccessSignInState(token));
      }
    } catch (e, stackTrace) {
      emit(ErrorSignInState(e, stackTrace));
      await _googleSignIn.logoutWithGoogle();
    }
  }
```

```dart 
 void _onSignUpWithEmailEvent(
      SignUpWithEmailEvent event, Emitter<SignInState> emit) async {
    try {
      emit(LoadingSignInState(true));
      final token = await _globalRepository.signUpWithEmail(
          DTOSignUpWithEmailRequest(
              email: event.mail, password: event.password, invite: ""));
      emit(LoadingSignInState(false));
      emit(SuccessSignInState(token, email: event.mail));
    } catch (e, stackTrace) {
      emit(LoadingSignInState(false));
      emit(ErrorSignInState(e, stackTrace));
    }
  }
```


<p align="center">
  <img src="https://github.com/GaLenN3228/wellai_mobile/blob/master/assets/google_auth.gif" alt="animated" width="300" height="600"/>
</p>


## Attach to doctor or nurse in calendar

The calendar screen is one of the most difficult widgets in this App. Because it contains a lot of requests and a difficult widgets structure. But we solve that problem by using slivers and bloc with loading state: 

```dart
    builder: (context, state) {
          if (state is LoadingTelehealthScheduleState) {
            return const LoadingIndicator();
          }
          if (state is DataTelehealthScheduleState) {
            return _DataStateBody(state: state);
          }
          return const SizedBox.shrink();
        },
```

And _DataStateBody

```dart
    return Scaffold(
      body: Column(
        children: [
          _TelehealthHeader(
            date: DateTime.now(),
          ),
          const SizedBox(height: 8),
          _DaysList(
            days: widget.state.schedulesDays,
            onChangeDay: (day) {
              setState(
                () {
                  selectedDayModel = day;
                },
              );
            },
          ),
          const SizedBox(height: 12),
          const Divider(
            color: ColorPalette.lightGray,
          ),
          selectedDayModel.times.isNotEmpty
              ? _ScheduleTimesList(day: selectedDayModel)
              : const _EmptyScheduleContianer()
        ],
      ),
    );
```

By tapping on date or time, we send request to server to get doctors/nurses list, which user can send message or request call. Sorting example: 

```dart
  _onInitialTelehealthScheduleEvent(InitialTelehealthScheduleEvent event,
      Emitter<TelehealthScheduleState> emit) async {
    try {
      final List<ScheduleDayModel> scheduleTime = [];

      final response = await _repository.getDocSchedule(
          DtoDocScheduleTelehealthRequest(
            start: tz.TZDateTime.now(_userStore.getCurrentTimeZone()),
            end: tz.TZDateTime.now(_userStore.getCurrentTimeZone()).add(
              const Duration(days: daysForSchedule),
            ),
          ),
      );

      scheduleTime.addAll(List.generate(daysForSchedule, (index) {
        final nowDate = DateTime.now();
        return ScheduleDayModel(
            DateTime(nowDate.year, nowDate.month, nowDate.day)
                .add(Duration(days: index)),
            []);
      }));

      final localizedWorkTime = <Worktime>[];

      for (var workTime in response.worktime) {
        final localizedTime = tz.TZDateTime(
          _userStore.getCurrentTimeZone(),
          workTime.work.date.year,
          workTime.work.date.month,
          workTime.work.date.day,
          workTime.time.hour,
          workTime.time.minute,
        ).toLocal();

        final localizedWork = workTime.work.copyWith(date: DateTime(localizedTime.year, localizedTime.month, localizedTime.day));
        localizedWorkTime.add(workTime.copyWith(time: localizedTime, work: localizedWork));
      }

      final groupedByDate = localizedWorkTime.groupListsBy(
            (element) => element.work.date,
      );
      groupedByDate.forEach((key, value) {
        scheduleTime.removeWhere((element) {
          return element.date.isAtSameMomentAs(key);
        });
        final groupedByTime = value.groupListsBy(
                (element) => element.time);
        final List<ScheduleTimeModel> times = [];
        groupedByTime.forEach((key, value) {
          times.add(ScheduleTimeModel(
              key,
              value
                  .map((e) => DoctorModel(
                      e.work.user.profile?.image?.name,
                      e.work.user.profile?.getFullName,
                      e.id,
                      e.work.date,
                      e.work.user.profile?.image?.blur))
                  .toList(),
              key.add(Duration(minutes: value.first.work.stepWorkTime))));
        });
        times.sort((a, b) => a.time.compareTo(b.time));
        scheduleTime.add(ScheduleDayModel(key, times));
      });
      scheduleTime.sort((a, b) => a.date.compareTo(b.date));
      emit(DataTelehealthScheduleState(
          scheduleTime, response.worktime.firstOrNull?.work.stepWorkTime ?? 0));
    } catch (e, stackTrace) {
      if (kDebugMode) {
        rethrow;
      }
      emit(ErrorTelehealthScheduleState(e, stackTrace));
    }
  }
```

<p align="center">
  <img src="https://github.com/GaLenN3228/wellai_mobile/blob/master/assets/calendar.gif" alt="animated" width="300" height="600"/>
</p>


## Profile editing 

The most common thing in the app like in 99% of all apps is the user profile editing feature. All patients are able to change their email, phone number and other credentials. Here is a simple event in User BLoC Which sends request to server, and changes user data.

```dart
  FutureOr<void> _buildDataChangesProfileEvent(
      DataChangesProfileEvent event, Emitter<EditProfileScreenState> emit) async {
    emit(LoadingEditProfileState(true));
    try {
      await _globalRepository.createEditProfile(
          event.profile, event.imagePath, event.isRemoveAvatar);
      final userInfo = await _globalRepository.getUserInfo();
      _userStore.upsertUser(userInfo);
      _hiveRepository.saveUserAvatar(
        userInfo.profile?.image?.name,
        userInfo.profile?.image?.blur,
      );
      emit(LoadingEditProfileState(false));
      emit(SuccessEditProfileScreenState());
    } catch (ex, stackTrace) {
      emit(LoadingEditProfileState(false));
      emit(ErrorEditProfileScreenState(ex, stackTrace));
    }
  }
```

<p align="center">
  <img src="https://github.com/GaLenN3228/wellai_mobile/blob/master/assets/profile_editing.gif" alt="animated" width="300" height="600"/>
</p>
