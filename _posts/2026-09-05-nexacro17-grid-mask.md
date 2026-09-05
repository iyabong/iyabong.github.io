---
title: "Nexacro17 그리드 maskedit"
date: 2026-09-05
categories: [Nexacro]
tags: [Nexacro17, Grid, maskedit]
---

# 넥사크로17 그리드에서 Float 입력 제대로 받기 (MaskEdit / displaytype / edittype)\

## 요구사항

1. 체크박스가 체크된 행만 입력상자 모양으로 보이고 편집 가능
2. `0`은 `0`으로 표시, `null`은 빈칸
3. 정수부 5자리, 소수부 5자리 제한
4. `.777` 입력 시 `0.777`로 저장·표시

## 최종 결과

[데이터셋]
![데이터셋](/assets/img/nexacro/2026-09-05-nexacro17-grid-mask-01.png)

[그리드표시]
![그리드표시](/assets/img/nexacro/2026-09-05-nexacro17-grid-mask-02.png)


```xml
 <Cell col="1" 
       text="bind:big" 
       displaytype="expr:chk == 1 ? (nexacro.isNumeric(big) ? 'maskcontrol' : 'editcontrol') : (nexacro.isNumeric(big) ? 'mask' : 'text')" 
       edittype="expr:chk == 1 ? 'mask' : 'none'" 
       maskedittype="number" 
       maskeditformat="####0.#####" 
       maskeditlimitbymask="both" textAlign="right"/>
```
컬럼 BIGDECIMAL

## format

- `#` 있으면 표시, `0` 강제
- `####0.#####` → .777 입력하면 0.777. 근데 null도 0으로 나옴
- `#####.#####` → null 빈칸. 근데 .777이 .777
- 그리드는 displaytype expr로 null이면 text/editcontrol로 빼서 해결. 마스크가 null 구분해주는 게 아님
  ※ 단독 MaskEdit은 onchanged에서 value==null이면 set_text("")

## displaytype / edittype

- display는 그림, edit는 동작
- editcontrol maskcontrol combocontrol 등 ~control은 모양만. edittype 없으면 클릭해도 아무것도 안 됨
- ~control도 expr로 지정됨
- maskcontrol은 maskedittype/maskeditformat 없이 그리면 박스 안 나올 수 있음
- editcontrol 좌측정렬 기본. textAlign="right" 주면 됨

