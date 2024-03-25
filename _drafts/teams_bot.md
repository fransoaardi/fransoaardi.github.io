ms teams bot 타당성 조사
ms teams bot 타당성 조사
1. ms teams 의 app 이란?
2. 만들어진 앱 사용
3. 직접 제작
4. (참고) 사내 hkmc teams app 사용
1. ms teams 의 app 이란?

ms teams 에 app 에 capability 가 3가지 있음

 tab
Tabs are Teams-aware webpages embedded in Microsoft Teams
ref: https://docs.microsoft.com/ko-kr/microsoftteams/platform/tabs/what-are-tabs
 bot
Bots in Microsoft Teams can be part of a one-to-one conversation, a group chat, or a channel in a Team
channel(스레드 기반 대화):
Your bot will also only have access to messages where it's @mentioned directly, although you can retrieve additional messages from the conversation using Microsoft Graph and elevated organization-level permissions.
groupchat(여러사람이 함께하는 대화):
 your bot will only have access to messages where it's @mentioned directly.
one-to-one chat: bot 과의 대화.
ref: https://docs.microsoft.com/ko-kr/microsoftteams/platform/bots/what-are-bots
messaging extension
message 로 부터 특정 action 을 정의하고, 결과를 메세지 등의 형태로 받을 수 있는것
jira 에 task 등록, 예약 후 예약 결과를 message 로 전달
action command: message 에서 trigger 가능
search command: message 에서 trigger 불가능
ref: https://docs.microsoft.com/ko-kr/microsoftteams/platform/messaging-extensions/what-are-messaging-extensions
Your bot will also only have access to messages where it's @mentioned directly

bot 에게 메세지를 전달하기 위해서는 mention 이 필수입니다.


2. 만들어진 앱 사용

Translateit (Bot 제공, messageExtension 제공하는것같음)

대화중에 @Translateit 으로 대화내용 전달하고, 결과를 해당 대화방에 받음
3. 직접 제작
가능한 방법:
 azure 에 web app bot 제작
C#, js(node.js), 템플릿 제공됨
 bot framework 이용 후, build 해서 연동
C#, js/ts(node.js), python 제공
ref: https://dev.botframework.com/
과정(web app bot 제작):
azure 에 js template 이용해서 web app bot 제작.
채널(ms teams 에서 사용할 수 있게) 연동
채널연동에서 링크 실행해서, ms teams 에서 사용
app market 에 등록되지 않음(검색이 안됨)
사용 권한 관련 설정이 안됨
teams workspace 연동된 azure 가 필요한것같음
teams 무료로는 더이상 테스트가 안됨
과정(bot framework 로 제작, azure 봇 채널 등록):
로컬에서 endpoint 마련함(server 로 옮길 수 있을것같음)
채널(ms teams 에서 사용하도록) 연동
manifest.json 내용 제작, zip 파일로 만들어서 '앱 > 사용자 지정 업로드' 하여 앱 등록
