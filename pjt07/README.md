# PJT 07

## B. Model

```python
#accounts/models.py
from django.db import models
from django.contrib.auth.models import AbstractUser

# Create your models here.
class User(AbstractUser):
    last_name = models.CharField(max_length=150)
    first_name = models.CharField(max_length=150)
    followers = models.ManyToManyField('self', symmetrical=False, related_name='followings')
```

```python
from django.db import models
from django.conf import settings

# Create your models here.
from django.db import models
from django.db.models.fields import related

class Review(models.Model):
    user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE)
    title = models.CharField(max_length=100)
    movie_title = models.CharField(max_length=50)
    like_users = models.ManyToManyField(settings.AUTH_USER_MODEL, related_name="like_review")
    rank = models.IntegerField()
    content = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

class Comment(models.Model):
    content = models.CharField(max_length=100)
    review = models.ForeignKey(Review, on_delete=models.CASCADE)
    user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE)
```



## C. Form

```python
# accounts/forms.py

from django import forms
from django.contrib.auth import get_user_model
from django.contrib.auth.forms import UserCreationForm

User = get_user_model()

class CustomUserCreationForm(UserCreationForm):
    class Meta:
        model = User
        fields = ('username', 'email', 'last_name', 'first_name')
```

```python
# community/forms.py

from django import forms
from .models import Review, Comment

class ReviewForm(forms.ModelForm):
    class Meta:
        model = Review
        fields = ('title', 'movie_title', 'rank', 'content',)


class CommentForm(forms.ModelForm):
    class Meta:
        model = Comment
        fields = ('content',)
```



## D. URL

```python
# pjt07/urls.py
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('accounts/', include('accounts.urls')),
    path('community/', include('community.urls')),
]
```

```python
# accounts/urls.py

from django.urls import path
from . import views

app_name = 'accounts'

urlpatterns = [
    path('signup/', views.signup, name='signup'),
    path('login/', views.login, name='login'),
    path('logout/', views.logout, name='logout'),
    path('<str:username>/', views.profile, name='profile'),
    path('<str:username>/follow/', views.follow, name='follow'),
]
```

```python
# community/urls.py

from django.urls import path
from . import views

app_name = 'community'

urlpatterns = [

    path('create/', views.create_review, name='create'),
    path('', views.review_index, name='index'),
    path('<int:review_pk>/', views.review_detail, name='detail'),
    path('<int:review_pk>/update/', views.update_review, name='update'),
    path('<int:review_pk>/delete/', views.delete_review, name='delete'),
    path('<int:review_pk>/like/', views.like_review, name='like'),
    path('<int:review_pk>/comments/create/', views.create_comment, name='comment'),
]
```



## E. Views & Templates

### i. ?????? ????????? ?????? ??? ??????

![image-20211022173416854](README.assets/image-20211022173416854.png)

```html
<!-- base.html -->
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta http-equiv="X-UA-Compatible" content="IE=edge">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Document</title>
</head>
<body>
  {% include '_nav.html' %}
  {% block content %}{% endblock content %}
</body>
</html>
```

```html
<!-- _nav.html -->
<nav>
  <a href="{% url 'community:index' %}">HOME</a>
  {% if request.user.is_authenticated %}
    <h1>??????????????? {{ request.user.username }}???</h1>
    <a href="{% url 'community:create' %}">?????? ??????</a>
    <a href="{% url 'accounts:logout' %}">????????????</a>
    <a href="{% url 'accounts:profile' request.user.username %}">?????????</a>
  {% else %}
    <a href="{% url 'accounts:signup' %}">????????????</a>
    <a href="{% url 'accounts:login' %}">?????????</a>
  {% endif %}
</nav>
```



### ii. ?????? ????????? ??????

![image-20211022173449292](README.assets/image-20211022173449292.png)

```python
# community/views.py
@login_required
@require_POST
def like_review(request, review_pk):
    review = get_object_or_404(Review, pk=review_pk)
    if review.like_users.filter(pk=request.user.pk).exists():
        review.like_users.remove(request.user)
    else:
        review.like_users.add(request.user)
    return redirect('community:detail', review.pk)
```

```html
<!-- community/detail.html -->
{% extends 'base.html' %}
{% block content %}
  <h1>{{ review.title }} : {{ review.movie_name }} </h1>
  {% if request.user in review.like_users.all %}
    <form action="{% url 'community:like' review.pk %}" method="POST">
    {% csrf_token %}
      <input type="submit" value="????????? ??????">
    </form>
  {% else %}
    <form action="{% url 'community:like' review.pk %}" method="POST">
    {% csrf_token %}
      <input type="submit" value="?????????">
    </form>
  {% endif %}
  <p>????????? ?????? : {{ review.like_users.all|length }}</p>
  <a href="{% url 'accounts:profile' review.user.username %}"><p>{{ review.user.username }}</p></a>
  <p>{{ review.content }}</p>
  <a href="{% url 'community:update' review.pk %}">??????</a>
  <a href="{% url 'community:delete' review.pk %}">??????</a>
  <a href="{% url 'community:index' %}">????????????</a>

  <hr>
  <h2>?????????</h2>
  {% for comment in comments %}
    <p>{{ comment.user }}</p>
    <p>{{ comment.content }}</p>
    <hr>
  {% endfor %}
  <form action="{% url 'community:comment' review.pk %}" method="POST">
    {% csrf_token %}
    {{ comment_form.as_p }}
    <input type="submit" value="?????? ??????">
  </form>

{% endblock content %}
```



### iii. ?????? ????????? ??????

![image-20211022173509747](README.assets/image-20211022173509747.png)

```python
# accounts/views.py
@require_safe
def profile(request, username):
    profile = get_object_or_404(User, username=username)

    context = {
        'profile': profile, 

    }
    return render(request, 'accounts/profile.html', context)

def follow(request, username):
    profile = get_object_or_404(get_user_model(), username=username)
    user = request.user
    if profile != user:
        if profile.followers.filter(pk=user.pk).exists():  
            profile.followers.remove(user)
        else:
            profile.followers.add(user)
    return redirect('accounts:profile', profile.username)
```

```html
<!-- accounts/profile -->
{% extends 'base.html' %}
{% block content %}
  <h1>{{ profile.username }}?????? ?????????</h1>
  <p>????????? ??? : {{ profile.followings.all|length }} / ????????? ??? : {{ profile.followers.all|length }}</p>
  {% if profile.username != request.user %}
    {% if request.user in profile.followers.all %}
      <form action="{% url 'accounts:follow' profile.username %}">
      {% csrf_token %}
      <input type="submit" value="????????????">
      </form>
    {% else %}
      <form action="{% url 'accounts:follow' profile.username %}">
      {% csrf_token %}
      <input type="submit" value="?????????">
      </form>
    {% endif %}

  {% else %}
  {% endif %}
{% endblock content %}
```



## ??????

views??? ???????????? user??? review?????? n:m ????????? ????????? ???????????? ???????????? ?????? ????????? ?????? ????????????. ????????? model??? schema??? ?????? ????????? ???????????? ????????? ???????????? ??? ????????? ??? ?????????. 

