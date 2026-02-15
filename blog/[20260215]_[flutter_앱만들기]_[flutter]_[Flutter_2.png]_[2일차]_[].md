# Flutter로_앱만들기_2일차

# 상황

현재 까진 로그인, 회원가입, 비밀번호 찾기, 메인 화면, 퀴즈 화면
이 구현이 완료되었다. 하지만 현재 버그들이 많아 수정중이다.

.
.
.

> *구조*

lib/
├── main.dart                 # 앱의 시작점, Firebase 초기화 및 라우팅 설정
├── models/                   # 데이터 구조 정의 (User, Quiz 등)
│   └── user_model.dart
├── screens/                  # 앱의 각 페이지 (UI 레이어)
│   ├── home_screen.dart      # 메인 대시보드 및 랭킹 표시
│   ├── login_screen.dart     # 이메일/구글 로그인
│   ├── register_screen.dart  # 회원가입 및 입력 검증
│   └── quiz_screen.dart      # 퀴즈 진행 및 결과 로직
├── services/                 # 비즈니스 로직 및 외부 통신 (Service 레이어)
│   ├── auth_service.dart     # Firebase Auth (가입, 로그인, 로그아웃)
│   ├── database_service.dart # Firestore CRUD (랭킹 스트림, 결과 저장)
│   └── level_service.dart    # 경험치 계산 및 레벨링 알고리즘
├── widgets/                  # 공통적으로 사용되는 재사용 위젯
│   ├── score_radar_chart.dart# 영역별 역량 분석 차트
│   └── custom_snackbar.dart  # 한글 에러 메시지 알림창



> *회원가입 버그*

회원가입을 할 때 이메일, 비밀번호를 'aaa'로 통일해서 보냈더니 회원가입 되었다는 알림은 뜨지만 막상 로그인을 해보니 회원가입이 되지 않는 버그가 발생했다.
그래서 firebase DB를 확인해 보니 'aaa' 데이터는 없었다. 문제를 확인해보니 firebase는 기본적으로 비밀번호 6자 미만 가입을 거부한다고 한다.
그래서 비밀번호를 6글자 미만으로 치고 가입 버튼을 눌렀을 때 알림창을 띄워 막는 로직을 추가하고 실제 에러가 났을 떄는 완료 창이 뜨지 않게 버그를 고쳤다. 

![버그_1](img/IMG_1299.PNG)
![버그_1](img/IMG_1300.PNG)

dart 코드:
```dart
// register_screen.dart 내 가입 버튼 로직
void _onSignUpPressed() async {
  final password = _pwController.text.trim();
  
  if (password.length < 6) {
    _showSnack("비밀번호는 최소 6자리 이상이어야 합니다.");
    return;
  }

  // AuthService를 통해 중복 이메일 체크 및 결과 수신
  final String? errorMessage = await _authService.signUpEmail(email, password, nickname);

  if (errorMessage == null) {
    _showSnack("회원가입 완료!", isError: false);
    Navigator.pop(context);
  } else {
    _showSnack(errorMessage); // "이미 사용 중인 이메일입니다" 등 출력
  }
}
```

> *랭킹 리스트 무한 로딩*

랭킹을 표시하기 위해 StreamBuilder와 Firestore의 orderBy를 사용했는데, 화면에는 인디케이터만 계속 돌고 데이터가 나타나지 않았습니다.


#### 원인
1. 복합 색인(Index) 미설정 혹은 오타: score와 createdAt 두 필드로 정렬을 수행할 때 Firestore는 반드시 인덱스가 필요합니다. 제 경우, 설정 중 createadAt이라는 오타가 있었습니다.

2. 데이터 누락: 특정 유저에게 정렬 기준이 되는 필드가 없으면 Firestore 쿼리는 해당 문서를 건너뛰거나 에러를 발생시킵니다.

#### 해결

1. Firebase 콘솔에서 잘못 생성된 인덱스를 삭제하고, 정확한 필드명으로 재설정했습니다.
2. 데이터 유실을 막기 위해 initializeUserData 함수에서 set의 merge: true 옵션을 사용하고, 랭킹 쿼리에 에러 핸들링을 추가

dart 코드:
```dart
// lib/services/database_service.dart
class DatabaseService {
  final FirebaseFirestore _db = FirebaseFirestore.instance;
  String? get uid => FirebaseAuth.instance.currentUser?.uid;

  // 유저 초기 데이터 생성 및 보완
  Future<void> initializeUserData(String email, String nickname) async {
    if (uid == null) return;

    await _db.collection('users').doc(uid).set({
      'uid': uid,
      'email': email,
      'nickname': nickname,
      'score': 0,
      'createdAt': FieldValue.serverTimestamp(), // 정확한 필드명 사용
    }, SetOptions(merge: true));
  }

  // 에러 발생 시 빈 리스트를 반환하여 로딩 멈춤 방지
  Stream<List<Map<String, dynamic>>> get rankingStream {
    return _db.collection('users')
        .orderBy('score', descending: true)
        .orderBy('createdAt', descending: false)
        .snapshots()
        .map((snapshot) => snapshot.docs.map((doc) => doc.data() as Map<String, dynamic>).toList())
        .handleError((error) {
          print("❌ 랭킹 로딩 에러: $error");
          return <Map<String, dynamic>>[];
        });
  }
}
```
.
.
.
