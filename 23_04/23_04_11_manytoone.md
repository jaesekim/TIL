# 23.04.11. Database N:1 복습

# N:1 (Comment - User)

**0개 이상의 댓글은 1개의 회원에 의해 작성 될 수 있다.**

## 플로우

1. `articles/models.py` 수정
    
    ```python
    class Comment(models.Model):
        user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE)  # 변경사항
        ...
    ```
    
    - `on_delete=models.CASCADE`: user가 삭제되면 comment도 삭제되도록 하는 옵션
    - `settings.AUTH_USER_MODEL`: User로 사용해도 되지만 이를 권장
2. `migration`진행
    - `python [manage.py](http://manage.py) makemigrations`
        
        NULL이면 안되는 필드인 ‘user’에 NULL값을 넣으려고 한다. 이를 해결하기 위해
        
        1) 기존에 넣을 디폴트값을 하나 주는 방법
        
        → `Please enter the default value now` : 현재 넣을 default value를 입력하면 된다.
        
        2) `models.py`에 가서 스스로 defalt 값을 설정
        
    - `python [manage.py](http://manage.py) migrate` : migrate 진행
    - `python [manage.py](http://manage.py) createsuperuser` : admin 계정 생성

### CREATE

- admin 계정으로 로그인 후 글 작성
- 글 작성하게 되면 User을 선택할 수 있게 되는 현상 발생
    
    ```python
    # articles/forms.py 수정
    
    class CommentForm(forms.ModelForm):
        class Meta:
            model = Comment
            # fields = '__all__'
            exclude = ('article', 'user',)  # exclude에 'user' 추가
    ```
    
- 수정 후 comment를 작성하면 에러 발생
    
    `NOT NULL constraint failed: articles_comment.user_id`
    
1. `articles/views.py` `comments_create`함수 수정
    - 요청한 user가 누구인지 알아야지 설정할 수 있다.
    - `request`라는 객체 안에 `request.user`라고 하면 알아서 세션을 가져와서 비교 후 일치하는 `user` 있으면 할당해 준다.
    
    ```python
    def comments_create(request, pk):
        article = Article.objects.get(pk=pk)
        comment_form = CommentForm(request.POST)
        if comment_form.is_valid():
            comment = comment_form.save(commit=False)
            comment.article = article
            comment.user = request.user   # 변경사항
            comment.save()
        return redirect('articles:detail', article.pk)
    ```
    
2. 댓글 작성자 보여주기
    
    `{{comment.content}} - {{ comment.user }}` - `detail.html` 추가
    

### DELETE

- 내 댓글을 다른 사람이 삭제하면 안되니까 관련해서 조건문 추가!

```python
# articles/views.py 수정

def comments_delete(request, pk, comment_pk):
    comment = Comment.objects.get(pk=comment_pk)
    if request.user == comment.user:
        comment.delete()
    return redirect('articles:detail', pk)
```

- 로그인을 하지 않으면 ‘삭제’버튼도 보이지 않도록 `detail.html`에서 수정

```html
{% if request.user == comment.user %}
	<form action="{% url 'articles:comments_delete' article.pk comment.pk%}" method="POST">
	  {% csrf_token %}
	  <input type="submit" value="삭제">
	</form>
{% endif %}
```

### 인증된 사용자에 대한 접근 제한

방법 1: 로그인 안한 사람이 댓글 작성하려 할 때 못하게 하기

방법 2: 로그인 안한 사람이 댓글 작성하려 할 때 로그인 페이지로 넘어가게 하기

- `.is_authenticated`
    - 인증된 사용자일 때만 로직을 처리할 수있도록 조건문을 달아준다.
    - 인증되지 않은 사용자일 때는 바로 return문으로 넘어가서 detail 페이지를 보여준다.
    
    ```python
    # 방법 1
    
    def comments_create(request, pk):
        if request.user.is_authenticated:   # if문에 추가
            article = Article.objects.get(pk=pk)
            comment_form = CommentForm(request.POST)
            if comment_form.is_valid():
                comment = comment_form.save(commit=False)
                comment.article = article
                comment.user = request.user
                comment.save()
        return redirect('articles:detail', article.pk)
    ```
    
    ```python
    # 방법 2
    
    def comments_create(request, pk):
        if request.user.is_authenticated:   # if문에 추가
            article = Article.objects.get(pk=pk)
            comment_form = CommentForm(request.POST)
            if comment_form.is_valid():
                comment = comment_form.save(commit=False)
                comment.article = article
                comment.user = request.user
                comment.save()
    		    return redirect('articles:detail', article.pk)
    		return redirect('accounts:login')
    ```
    
- 로그인 안 한 사람들에게는 CommentForm 자체도 볼 수 없게 하기
    
    ```html
    {% if comments.user == request.user %}
    	<form action="{% url 'articles:comments_create' article.pk %}" method="POST">
    	  {% csrf_token %}
    	  {{comment_form}}
    	  <input type="submit" value="저장">
    	</form>
    {% else %}
      <a href="{% url 'accounts:login' %}">[댓글을 작성하려면 로그인하세요]</a>
    {% endif %}
    ```