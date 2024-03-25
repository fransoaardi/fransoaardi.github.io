---
layout: post
title: 노트북 Sleep 시 WiFi 연결해제 방지하는 방법
date: 2022-06-17 19:00:00 +0900
categories: [Wiki]
tags: [wifi, windows]
toc: true
comments: true
---

# Introduction

업무용 노트북을 Wi-Fi 에 연결해두고 점심을 먹든 하는 이유로 장시간 후에 sleep 혹은 pending 상태에 들어가는 경우 와이파이가 끊어져서 많은 작업들이 엉망이 되어있는 경우들이 있다. 노트북은 전력의 효율적인 이용을 위해 power management 의 일환으로 Wi-Fi 네트워크 카드의 전원을 차단한다고 한다. 장치관리자(Device Manager) 에서 네트워크 카드를 확인하고 설정에서 설정을 변경하면 해결이 된다고 하는데, 이 메뉴가 보이지 않았는데 이를 보이게 하는 registry 변경 매뉴얼을 전달받아 공유한다. 

# Contents 

- `cmd.exe` 를 관리자 권한으로 실행한다.
- `reg add HKLM\System\CurrentControlSet\Control\Power /v PlatformAoAcOverride /t REG_DWORD /d 0` 를 입력한다.
- 재 부팅후 네트워크 드라이브에 power management 탭 활성화 되어있는지 확인한다.
- power management 탭 활성화 되어있으면 해당 옵션(`Allow the computer to turn off the device to save power`) 비활성 처리한다.
- power management 탭 활성화 안되어있으면 `$ reg delete  "HKLM\System\CurrentControlSet\Control\Power" /v PlatformAoAcOverride` 를 입력한다. 
- 재 부팅후 네트워크 드라이브에 power management 탭 활성화 되어있는지 확인하고 위의 옵션을 비활성 처리한다.