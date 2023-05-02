# 23.04.12.  Database M:N

**들어가기 전에**

- django에서 shell plus를 사용하고 싶으면 django extensions를 넣어줘야 한다.
- pip와 [settings.py](http://settings.py) 안의 INSTALLED_APPS 안에 `‘django_extensions’` 추가 필요
- 테이블을 구현하고 안에 이미 데이터가 많이 들어가고 이러면 에러가 나서 그냥 싹다 날려버리는 게 마음 편하다

# Shell_Plus 구현

```python
doctor1 = Doctor.objects.create(name='alice')

patient1 = Patient.objects.create(name="carol", doctor=doctor1)
```

## N:1의 한계

환자에게 FK값으로 의사 번호를 할당할 때의 한계점

1번 환자가 서로 다른 의사 두 명에게 방문하려한다고 가정

| id | name |
| --- | --- |
| 1 | alice |
| 2 | bella |

| id | name | doctor_id |
| --- | --- | --- |
| 1 | carol | 1 |
| 2 | dane | 2 |
| 3 | carol | 2 |

**정규화에 어긋난다.**

따라서 예약 테이블을 따로 만드는 방법을 사용한다.

- `hospitals/models.py` 클래스 선언
    
    ```python
    class Doctor(models.Model):
        name = models.TextField()
    
        def __str__(self):
            return f"{self.name} 전문의"
    
    class Patient(models.Model):
        # doctor = models.ForeignKey(Doctor, on_delete=models.CASCADE)
        name = models.TextField()
    
        def __str__(self):
            return f"{self.pk}번 환자 {self.name}"
        
    
    # 중개 테이블
    class Reservation(models.Model):
        doctor  = models.ForeignKey(Doctor, on_delete=models.CASCADE)
        patient = models.ForeignKey(Patient, on_delete=models.CASCADE)
    
        def __str__(self):
            return f"{self.doctor_id}번 의사의 {self.patient_id}번 환자"
    ```
    
    model관계
    
    Reservation → Doctor or Patient : 정참조
    
    Doctor or Patient → Reservation : 역참조
    

### `shell plus`로 확인

- 중개테이블 정보 확인
    
    ```python
    doctor1 = Doctor.objects.create(name='alice')
    
    patient1 = Patient.objects.create(name='carol')
    
    Reservation.objects.create(doctor=doctor1, patient=patient1)
    # <Reservation: 1번 의사의 1번 환자>
    
    # 의사 정보로 예약 정보 찾기(역참조)
    doctor1.reservation_set.all()
    # 환자 정보로 예약 정보 찾기(역참조)
    patient1.reservation_set.all()
    ```
    
- 1번 의사에게 새로운 환자 예약이 생성된다면
    
    ```python
    patient2 = Patient.objects.create(name='dane')
    
    Reservation.objects.create(doctor=doctor1, patient=patient2)
    
    doctor1.reservation_set.all()
    # <QuerySet [<Reservation: 1번 의사의 1번 환자>, <Reservation: 1번 의사의 2번 환자>]>
    ```
    

---

# Many to Many Field

django에서는 M:N 관계를 많이 사용하는 것을 알고 자동으로 중개 테이블을 만들어주는 기능이 있다.

```python
class Patient(models.Model):
    doctors = models.ManyToManyField(Doctor)  
		# 변경사항: ForeignKey가 아니라 ManyToManyField(M:N 관계 맺을 테이블) 사용.
    name = models.TextField()

    def __str__(self):
        return f"{self.pk}번 환자 {self.name}"
```

- db를 확인해보면 django가 새로이 중개 테이블을 생성해 줬다.

```python
# shell_plus
# 환자, 의사 추가

doctor1 = Doctor.objects.create(name='alice')
patient1 = Patient.objects.create(name='carol')
patient2 = Patient.objects.create(name='dane')

# 환자 1번을 의사 1번에게 추가해주는 명령
patient1.doctors.add(doctor1)

# 환자 1번에 대한 모든 의사들을 조회
patient1.doctors.all()

```

- 의사에서 예약 목록 역참조
    - 방법 1
    `doctor1.paient_set.all()`
    - 방법 2
        
        ```python
        class Patient(models.Model):
            doctors = models.ManyToManyField(Doctor, related_name='patients')  
        		# 변경사항: ForeignKey 내 related_name 설정을 통해 역참조 명령어 갱신
            name = models.TextField()
        ```
        
        ```python
        doctor1 = Doctor.objects.get(id=1)  # doctor1 가져오기
        
        doctor1.patients.all()  # 바뀐 명렁어로 역참조
        ```
        
- 예약 취소하기(삭제)
    
    기존에는 해당하는 Reservation을 찾아서 지워야했다면 이제는 `.remove()` 사용.
    
    ```python
    # shell plus
    patient1 = Patient.objects.get(id=1)  # 환자 1번 불러오기
    
    patient1
    # <Patient: 1번 환자 carol>
    
    doctor1.patients.remove(patient1)
    # doctor1이 patient1 진료 예약 취소
    ```
    

---

# 중개 테이블 수동 생성

중개 테이블에 추가 데이터를 사용해서 다대다 관계와 연결하는 이유 등으로 수동으로 만들 경우가 있다.

```python
class Reservation(models.Model):
    doctor  = models.ForeignKey(Doctor, on_delete=models.CASCADE)
    patient = models.ForeignKey(Patient, on_delete=models.CASCADE)
    
    symptom = models.TextField()
    reserved_at = models.DateTimeField(auto_now_add=True)

# 다음과 같이 doctor, patient 두 테이블의 단순 연결 뿐만 아니라 추가적인 다른
# 데이터들을 테이블에 저장해야 할 경우 중개 테이블을 직접 만들어 줘야 한다.
```

## `through` 옵션

- 수동으로 중개 테이블을 만들고 ManyToManyField의 장점을 같이 사용하고 싶을 때 사용

```python
class Patient(models.Model):
    doctors = models.ManyToManyField(Doctor, through='Reservation')
		# through 옵션을 준다
    name = models.TextField()

    def __str__(self):
        return f"{self.pk}번 환자 {self.name}"
```

### 플로우

```python
# shell_plus
# 환자, 의사 추가

doctor1 = Doctor.objects.create(name='alice')
patient1 = Patient.objects.create(name='carol')
patient2 = Patient.objects.create(name='dane')

# 방법 1
reservation1 = Reservation(doctor=doctor1, patient=patient1, symptom='headache')
reservation1.save()

# 방법2
patient2.doctors.add(doctor1, through_defaults={'symptom': 'flu'})
```

---

# ManyToManyField 활용하기

## ManyToManyField

- ManyToManyField(to, **options)
- 다대다 관계 설정 시 사용하는 모델 필드
- 하나의 필수 위치인자(M:N 관계로 설정할 모델 클래스)가 필요
- 모델 필드의 RelatedManager를 사용하여 관련 개체를 추가, 제거 또는 만들 수 있다.
    - `add()`, `remove()`, `create()`, `clear()` …
- `db_table` arguments를 사용해서 중개 테이블의 이름을 변경할 수도 있다.
- `symmetrical`: 한 쪽이 일촌신청을 넣고 상대가 수락하면 양 쪽 다 연결되는 느낌. `default=False`

---

# 좋아요 기능

```python
class Article(models.Model):
    user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE)
    like_users = models.ManyToManyField(settings.AUTH_USER_MODEL)
		...

'''
ERRORS:
articles.Article.like_users: (fields.E304) Reverse accessor for 'articles.Article.like_users' 
clashes with reverse accessor for 'articles.Article.user'.
'''
```

- user의 역참조 manager와 like_users의 역참조 manager가 같아서 충돌이 일어났다.
    
    `.article_set`을 구분할 수 없기 때문이다.
    
- 따라서 겹치는 것들 중 최소 하나는 이름을 바꿔줘야 한다.

`like_users = models.ManyToManyField(settings.AUTH_USER_MODEL, related_name='like_articles')`

## like 구현

1. [`urls.py`](http://urls.py)
    
    `path('int:article_pk/likes/', views.likes, name='likes'),` 추가
    
2. `views.py`
    
    ```python
    def likes(request, article_pk):
        article = Article.objects.get(pk=article_pk)
        # 좋아요 버튼 이미 눌렀으면 좋아요 취소
        if article.like_users.filter(pk=request.user.pk).exists():
            # request를 보낸 유저는 이미 좋아요를 눌렀던 유저
            article.like_users.remove(request.user)
            # 좋아요 취소
        # 좋아요 버튼 안 눌렀었다면 좋아요
        else:
            article.like_users.add(request.user)
        return redirect('articles:index')
    ```
    
3. `index.html`
    - 좋아요 개수 구현
    
    ```html
    <span>
      좋아요: {{article.like_users.count}} 개
    </span>
    ```
    
    - 좋아요 버튼 구현
    
    ```html
    {% for article in articles %}
        <p>
          [{{article.id}}] 
          <a href="{% url 'articles:detail' article.pk %}" id="article-title">{{article.title}}</a>
          - 작성자: {{article.user}}
    
        </p>
        <form action="{% url 'articles:likes' article.pk %}" method="POST">
          {% csrf_token %}
          {% if request.user in article.like_users.all %}
            <input type="submit" value="좋아요 취소">
          {% else %}
            <input type="submit" value="좋아요">
          {% endif %}
        </form>
        <hr />
      {% endfor %}
    ```