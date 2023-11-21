---
title: Front Matter 스니펫 기능
date : 2023-11-21 21:00
category : [Blog, Markdown]
tags : [vscode, markdown, front matter, code snippet]
---

## 발단
markdown 게시물마다 Front Matter를 써주는 것은 너무나 귀찮은 일이었다. 
포맷은 일정하니 빠르고 편하게 입력하는 방법이 없을까? 

## 코드 스니펫
패키지의 메서드를 쓸 때처럼 VS Code에서 제공중인 `코드 스니펫` 기능을 사용하면 되겠지 싶었다. 생각보다 쉽다..! 유용하게 사용할 듯

---
## 방법
1. 설정 - 사용자 코드 조각 구성(코드 스니펫)
![](/assets/img/YY-MM/2023-11-21-21-25-01.png)<br>

2. markdown.json으로 이동
![](/assets/img/YY-MM/2023-11-21-21-27-30.png)

3. frontMatter 스니펫 작성
``` json
"Front Matter Generate" : { // 이름
		"prefix": "front_matter", // body를 불러오는 글자
		"body": [ // 불러와지는 코드 내용
			"---",
			"title: ",
			"date : ",
			"category : []",
			"tags : []",
			"---"
		],
		"description": "Front Matter" // 스니펫 설명
	}
```

4. `cmd + P`를 누르고 settings.json에서 markdown에 대해 snippet 관련 설정을 해준다.
``` json
 "[markdown]" :  {
        "editor.quickSuggestions": true,
        "editor.snippetSuggestions": "top"
    }
```
4. 결과
![](/assets/img/YY-MM/2023-11-21-21-31-29.png)

markdown에서 뿐만 아니라 여러모로 잘 사용할 수 있을 것 같다!