# QUIPU Renewal Project Recruit

## Overview
QUIPU Renewal Project는 **QUIPU Main Web 2025**와 **QUIPU Admin Web**를 함께 개선하는 단기 리뉴얼 프로젝트입니다.  
이번 리뉴얼의 핵심은 운영 효율을 높이고, 매년 바뀌는 동아리 운영 방향을 웹 서비스에 유연하게 반영할 수 있는 구조를 만드는 것입니다.

## Current State
### Main Web
- 메인 웹의 주요 콘텐츠가 하드코딩되어 있어, 내용 수정 시 코드 변경과 재배포가 필요함
- 모집 폼이 2025년 기준 포맷에 고정되어 있어, 해마다 달라지는 모집 항목과 운영 방향을 반영하기 어려움

### Admin Web
- 단일 사전 발급 비밀번호 방식으로 운영 중
- 비밀번호 발급/수정 등 계정·권한 관리 로직이 부족함

## Renewal Goals
### 1) Content Management Integration
- Admin Web에서 Main Web 콘텐츠를 수정할 수 있도록 기능 추가
- 운영진이 코드 수정 없이도 공지·소개·콘텐츠를 관리할 수 있는 구조로 전환

### 2) Dynamic Recruitment Form
- Admin Web에서 모집 폼을 자유롭게 구성할 수 있도록 개선
- 매년 달라지는 동아리 활동 방향과 모집 기준을 빠르게 반영 가능하도록 설계

### 3) Access Control Improvement
- Admin Web의 접근 권한 로직을 보완
- 비밀번호/권한 관리 체계를 강화해 운영 안정성 확보

### 4) Main Web Feature Expansion
- Main Web 신규 기능 도입
- 예시: 자유 게시판, 질문 게시판, 미니 게임 등 커뮤니티형 기능

### 5) Admin Web Feature Expansion
- 운영 데이터 확인 기능 고도화
- 예시: 회원의 성별·나이·학과 등 기본 통계 및 시각화 기능

## Timeline
- **개발 기간:** 3월 16일 ~ 4월 5일 (약 3주)

## Tech Stack (Tentative)
- **Frontend:** React, TypeScript
- **Backend:** Express, TypeScript
- **Database:** MongoDB
- **Deployment:** Vercel 또는 AWS 또는 온프레미스 (상황에 따라 결정)

## Participation Requirements
- 약 3주간 진행되는 단기 프로젝트로, **주 2~3일 이상 참여 가능한 일정 확보**가 필요함
- 개발이 처음이어도 참여 가능
  - 문법 자체를 처음부터 강의하는 방식보다는, 개발 방향을 공유하고 AI 기반 바이브 코딩 중심으로 진행
- **ChatGPT Plus 또는 Cursor Pro 구독 필수** (약 월 3만 원)
- GitHub 기본 사용법은 각자 학습(유튜브 등) 후 진행, 막히는 부분은 질문 가능
- 가장 중요한 조건은 기술 숙련도보다 **흥미, 질문하는 태도, 끝까지 해결하려는 의지**

## Working Process
- 주 1~2회 비대면 회의 진행 (매주 일정 조율)
- 회의 방식: 카메라 필수 아님, 마이크 필수
- 각자 구현 기능 브리핑 + 다음 개발 범위/우선순위 결정
- GitHub Pull Request 기반 협업 및 코드 리뷰 진행
- 리뷰 피드백을 반영하며 점진적으로 품질을 높이는 방식으로 운영