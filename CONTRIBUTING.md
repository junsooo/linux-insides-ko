기여하기
================================================================================

[linux-insides-ko](https://github.com/junsooo/linux-insides-ko)에 기여하기 위해서 아래 규칙들을 따라주시면 감사하겠습니다.

1. 레포지토리 오른쪽 상단의 fork 버튼을 눌러주세요.

    ![fork](http://oi58.tinypic.com/jj2trm.jpg)

2. 본인의 계정으로 레포지토리를 clone 해주세요.

    ```
    git clone git@github.com:your_github_username/linux-insides.git
    ```

3. 새로운 브랜치를 만들어주세요.

    ```
    git checkout -b "linux-bootstrap-1-fix"
    ```
    이름을 붙이고 싶은대로 붙여주세요.

4. 특정 챕터에 대해 번역자가 겹치는 일을 막기 위해, 이슈를 남겨서 다른 사람들이 알 수 있도록 해주세요.
    
    예를 들어 DataStructures의 linux-datastructures-1.md 파일을 번역하고 싶으면
    이슈 제목을 "DataStructures/linux-datastructures-1 번역"으로 지정해주세요.
    
    ![이슈](https://github.com/junsooo/linux-insides-ko/blob/master/issue_example.PNG)

5. 번역을 마음껏 진행해주세요.

6. `korean-contributors.md`에 본인의 이름을 꼭 넣어주세요.

7. 번역에 대해 commit을 하고 push한 이후에, Github에서 풀 리퀘스트를 보내주세요.

**중요**

번역을 진행하는 도중에 다른 사람의 풀 리퀘스트가 merge 되면 `master` 브랜치의 내용이 바뀌므로 자신의 fork한 레포지토리와 충돌할 수 있습니다. 따라서 번역을 진행하고 해당 내용을 push 하기 전에 반드시 `master` 브랜치에 대해 `git pull`을 진행해 `master` 브랜치와 충돌이 나지 않도록 해주세요. 

감사합니다.
