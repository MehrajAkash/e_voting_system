# Django Online Voting System — Full Code

---

## 1. Project Setup Commands

```bash
pip install django
django-admin startproject voting_system
cd voting_system
python manage.py startapp voting
```

---

## 2. voting_system/settings.py (add app)

```python
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'voting',  # Add this
]

LOGIN_URL = '/login/'
LOGIN_REDIRECT_URL = '/vote/'
LOGOUT_REDIRECT_URL = '/'
```

---

## 3. voting/models.py

```python
from django.db import models
from django.contrib.auth.models import User


class Election(models.Model):
    title = models.CharField(max_length=200)
    description = models.TextField(blank=True)
    is_active = models.BooleanField(default=True)
    start_date = models.DateTimeField()
    end_date = models.DateTimeField()
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return self.title


class Candidate(models.Model):
    election = models.ForeignKey(Election, on_delete=models.CASCADE, related_name='candidates')
    name = models.CharField(max_length=100)
    party = models.CharField(max_length=100, blank=True)
    description = models.TextField(blank=True)

    def __str__(self):
        return f"{self.name} ({self.election.title})"

    def vote_count(self):
        return self.votes.count()


class Vote(models.Model):
    voter = models.ForeignKey(User, on_delete=models.CASCADE)
    candidate = models.ForeignKey(Candidate, on_delete=models.CASCADE, related_name='votes')
    election = models.ForeignKey(Election, on_delete=models.CASCADE)
    voted_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        unique_together = ('voter', 'election')  # One vote per user per election

    def __str__(self):
        return f"{self.voter.username} voted in {self.election.title}"
```

---

## 4. voting/forms.py

```python
from django import forms
from django.contrib.auth.forms import UserCreationForm
from django.contrib.auth.models import User
from .models import Vote


class RegisterForm(UserCreationForm):
    email = forms.EmailField(required=True)
    first_name = forms.CharField(max_length=50)
    last_name = forms.CharField(max_length=50)

    class Meta:
        model = User
        fields = ['username', 'first_name', 'last_name', 'email', 'password1', 'password2']


class VoteForm(forms.Form):
    candidate = forms.ModelChoiceField(
        queryset=None,
        widget=forms.RadioSelect,
        empty_label=None
    )

    def __init__(self, election, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.fields['candidate'].queryset = election.candidates.all()
```

---

## 5. voting/views.py

```python
from django.shortcuts import render, redirect, get_object_or_404
from django.contrib.auth import login, logout, authenticate
from django.contrib.auth.decorators import login_required
from django.contrib.auth.forms import AuthenticationForm
from django.contrib import messages
from django.utils import timezone
from .models import Election, Candidate, Vote
from .forms import RegisterForm, VoteForm


def home(request):
    elections = Election.objects.filter(is_active=True)
    return render(request, 'voting/home.html', {'elections': elections})


def register_view(request):
    if request.user.is_authenticated:
        return redirect('vote_list')
    if request.method == 'POST':
        form = RegisterForm(request.POST)
        if form.is_valid():
            user = form.save()
            login(request, user)
            messages.success(request, 'Registration successful! You can now vote.')
            return redirect('vote_list')
    else:
        form = RegisterForm()
    return render(request, 'voting/register.html', {'form': form})


def login_view(request):
    if request.user.is_authenticated:
        return redirect('vote_list')
    if request.method == 'POST':
        form = AuthenticationForm(data=request.POST)
        if form.is_valid():
            user = form.get_user()
            login(request, user)
            messages.success(request, f'Welcome back, {user.first_name or user.username}!')
            return redirect('vote_list')
        else:
            messages.error(request, 'Invalid username or password.')
    else:
        form = AuthenticationForm()
    return render(request, 'voting/login.html', {'form': form})


def logout_view(request):
    logout(request)
    messages.info(request, 'You have been logged out.')
    return redirect('home')


@login_required
def vote_list(request):
    now = timezone.now()
    elections = Election.objects.filter(is_active=True, start_date__lte=now, end_date__gte=now)
    voted_election_ids = Vote.objects.filter(voter=request.user).values_list('election_id', flat=True)
    return render(request, 'voting/vote_list.html', {
        'elections': elections,
        'voted_election_ids': voted_election_ids,
    })


@login_required
def vote(request, election_id):
    election = get_object_or_404(Election, id=election_id, is_active=True)
    now = timezone.now()

    if now < election.start_date or now > election.end_date:
        messages.error(request, 'This election is not currently open.')
        return redirect('vote_list')

    if Vote.objects.filter(voter=request.user, election=election).exists():
        messages.warning(request, 'You have already voted in this election.')
        return redirect('results', election_id=election.id)

    if request.method == 'POST':
        form = VoteForm(election, request.POST)
        if form.is_valid():
            candidate = form.cleaned_data['candidate']
            Vote.objects.create(voter=request.user, candidate=candidate, election=election)
            messages.success(request, f'Your vote for {candidate.name} has been recorded!')
            return redirect('results', election_id=election.id)
    else:
        form = VoteForm(election)

    return render(request, 'voting/vote.html', {'election': election, 'form': form})


def results(request, election_id):
    election = get_object_or_404(Election, id=election_id)
    candidates = election.candidates.all().order_by('-votes__id')

    candidate_data = []
    total_votes = Vote.objects.filter(election=election).count()

    for candidate in election.candidates.all():
        count = candidate.vote_count()
        percentage = round((count / total_votes * 100), 1) if total_votes > 0 else 0
        candidate_data.append({
            'candidate': candidate,
            'count': count,
            'percentage': percentage,
        })

    candidate_data.sort(key=lambda x: x['count'], reverse=True)

    return render(request, 'voting/results.html', {
        'election': election,
        'candidate_data': candidate_data,
        'total_votes': total_votes,
    })
```

---

## 6. voting/admin.py

```python
from django.contrib import admin
from .models import Election, Candidate, Vote


class CandidateInline(admin.TabularInline):
    model = Candidate
    extra = 2


@admin.register(Election)
class ElectionAdmin(admin.ModelAdmin):
    list_display = ['title', 'is_active', 'start_date', 'end_date', 'total_votes']
    list_editable = ['is_active']
    inlines = [CandidateInline]

    def total_votes(self, obj):
        return Vote.objects.filter(election=obj).count()
    total_votes.short_description = 'Total Votes'


@admin.register(Candidate)
class CandidateAdmin(admin.ModelAdmin):
    list_display = ['name', 'party', 'election', 'vote_count']


@admin.register(Vote)
class VoteAdmin(admin.ModelAdmin):
    list_display = ['voter', 'candidate', 'election', 'voted_at']
    readonly_fields = ['voter', 'candidate', 'election', 'voted_at']
```

---

## 7. voting/urls.py

```python
from django.urls import path
from . import views

urlpatterns = [
    path('', views.home, name='home'),
    path('register/', views.register_view, name='register'),
    path('login/', views.login_view, name='login'),
    path('logout/', views.logout_view, name='logout'),
    path('vote/', views.vote_list, name='vote_list'),
    path('vote/<int:election_id>/', views.vote, name='vote'),
    path('results/<int:election_id>/', views.results, name='results'),
]
```

---

## 8. voting_system/urls.py

```python
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('voting.urls')),
]
```

---

## 9. Templates

### voting/templates/voting/base.html

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{% block title %}Online Voting System{% endblock %}</title>
    <style>
        * { box-sizing: border-box; margin: 0; padding: 0; }
        body { font-family: 'Segoe UI', sans-serif; background: #f0f4f8; color: #2d3748; }
        nav { background: #1a365d; padding: 1rem 2rem; display: flex; justify-content: space-between; align-items: center; }
        nav a { color: white; text-decoration: none; margin-left: 1rem; font-weight: 500; }
        nav .brand { font-size: 1.3rem; font-weight: bold; color: #90cdf4; }
        .container { max-width: 900px; margin: 2rem auto; padding: 0 1rem; }
        .card { background: white; border-radius: 12px; padding: 2rem; box-shadow: 0 2px 10px rgba(0,0,0,0.08); margin-bottom: 1.5rem; }
        .btn { display: inline-block; padding: 0.6rem 1.4rem; border-radius: 8px; border: none; cursor: pointer; font-size: 1rem; text-decoration: none; }
        .btn-primary { background: #2b6cb0; color: white; }
        .btn-success { background: #276749; color: white; }
        .btn-danger { background: #c53030; color: white; }
        .btn:hover { opacity: 0.9; }
        .alert { padding: 0.8rem 1.2rem; border-radius: 8px; margin-bottom: 1rem; }
        .alert-success { background: #c6f6d5; color: #276749; }
        .alert-error { background: #fed7d7; color: #c53030; }
        .alert-info { background: #bee3f8; color: #2b6cb0; }
        .alert-warning { background: #fefcbf; color: #744210; }
        input[type=text], input[type=email], input[type=password] {
            width: 100%; padding: 0.7rem 1rem; border: 1px solid #cbd5e0;
            border-radius: 8px; font-size: 1rem; margin-bottom: 1rem;
        }
        label { display: block; font-weight: 600; margin-bottom: 0.3rem; color: #4a5568; }
        h1, h2 { color: #1a365d; margin-bottom: 1rem; }
    </style>
</head>
<body>
<nav>
    <span class="brand">🗳️ VotePortal</span>
    <div>
        <a href="{% url 'home' %}">Home</a>
        {% if user.is_authenticated %}
            <a href="{% url 'vote_list' %}">Elections</a>
            <a href="{% url 'logout' %}">Logout ({{ user.username }})</a>
        {% else %}
            <a href="{% url 'login' %}">Login</a>
            <a href="{% url 'register' %}">Register</a>
        {% endif %}
    </div>
</nav>
<div class="container">
    {% for message in messages %}
        <div class="alert alert-{{ message.tags }}">{{ message }}</div>
    {% endfor %}
    {% block content %}{% endblock %}
</div>
</body>
</html>
```

---

### voting/templates/voting/home.html

```html
{% extends 'voting/base.html' %}
{% block title %}Home - VotePortal{% endblock %}
{% block content %}
<div class="card" style="text-align:center; background: linear-gradient(135deg, #1a365d, #2b6cb0); color: white;">
    <h1 style="color:white; font-size:2rem;">🗳️ Online Voting System</h1>
    <p style="margin: 1rem 0; font-size: 1.1rem;">Secure, transparent, and easy voting for everyone.</p>
    {% if not user.is_authenticated %}
        <a href="{% url 'register' %}" class="btn btn-success" style="margin-right:1rem;">Register</a>
        <a href="{% url 'login' %}" class="btn btn-primary">Login to Vote</a>
    {% else %}
        <a href="{% url 'vote_list' %}" class="btn btn-success">Go to Elections →</a>
    {% endif %}
</div>

<h2>Active Elections</h2>
{% for election in elections %}
<div class="card">
    <h3>{{ election.title }}</h3>
    <p style="color:#718096; margin-bottom:1rem;">{{ election.description }}</p>
    <small>📅 {{ election.start_date|date:"M d, Y" }} – {{ election.end_date|date:"M d, Y" }}</small><br><br>
    <a href="{% url 'results' election.id %}" class="btn btn-primary">View Results</a>
</div>
{% empty %}
<div class="card"><p>No active elections at the moment.</p></div>
{% endfor %}
{% endblock %}
```

---

### voting/templates/voting/register.html

```html
{% extends 'voting/base.html' %}
{% block title %}Register{% endblock %}
{% block content %}
<div class="card" style="max-width: 500px; margin: 0 auto;">
    <h2>📝 Create Account</h2>
    <form method="post">
        {% csrf_token %}
        {% for field in form %}
            <label>{{ field.label }}</label>
            {{ field }}
            {% if field.errors %}
                <p style="color:red; font-size:0.85rem;">{{ field.errors|join:", " }}</p>
            {% endif %}
        {% endfor %}
        <button type="submit" class="btn btn-primary" style="width:100%;">Register</button>
    </form>
    <p style="margin-top:1rem; text-align:center;">Already have an account? <a href="{% url 'login' %}">Login</a></p>
</div>
{% endblock %}
```

---

### voting/templates/voting/login.html

```html
{% extends 'voting/base.html' %}
{% block title %}Login{% endblock %}
{% block content %}
<div class="card" style="max-width: 400px; margin: 0 auto;">
    <h2>🔐 Login</h2>
    <form method="post">
        {% csrf_token %}
        {% for field in form %}
            <label>{{ field.label }}</label>
            {{ field }}
        {% endfor %}
        <button type="submit" class="btn btn-primary" style="width:100%; margin-top:0.5rem;">Login</button>
    </form>
    <p style="margin-top:1rem; text-align:center;">No account? <a href="{% url 'register' %}">Register here</a></p>
</div>
{% endblock %}
```

---

### voting/templates/voting/vote_list.html

```html
{% extends 'voting/base.html' %}
{% block title %}Elections{% endblock %}
{% block content %}
<h2>🗳️ Active Elections</h2>
{% for election in elections %}
<div class="card">
    <h3>{{ election.title }}</h3>
    <p style="color:#718096;">{{ election.description }}</p>
    <p><small>📅 {{ election.start_date|date:"M d, Y H:i" }} – {{ election.end_date|date:"M d, Y H:i" }}</small></p>
    <br>
    {% if election.id in voted_election_ids %}
        <span style="color:green; font-weight:bold;">✅ You have voted</span>
        <a href="{% url 'results' election.id %}" class="btn btn-primary" style="margin-left:1rem;">View Results</a>
    {% else %}
        <a href="{% url 'vote' election.id %}" class="btn btn-success">Vote Now</a>
    {% endif %}
</div>
{% empty %}
<div class="card"><p>No elections are currently open.</p></div>
{% endfor %}
{% endblock %}
```

---

### voting/templates/voting/vote.html

```html
{% extends 'voting/base.html' %}
{% block title %}Vote - {{ election.title }}{% endblock %}
{% block content %}
<div class="card">
    <h2>🗳️ {{ election.title }}</h2>
    <p style="color:#718096; margin-bottom:1.5rem;">{{ election.description }}</p>
    <form method="post">
        {% csrf_token %}
        <p style="font-weight:600; margin-bottom:1rem;">Select your candidate:</p>
        {% for radio in form.candidate %}
        <div style="display:flex; align-items:center; padding:0.8rem; border:1px solid #e2e8f0; border-radius:8px; margin-bottom:0.5rem; cursor:pointer;">
            {{ radio.tag }}
            <label for="{{ radio.id_for_label }}" style="margin-left:0.8rem; cursor:pointer; font-weight:normal;">
                <strong>{{ radio.choice_label }}</strong>
            </label>
        </div>
        {% endfor %}
        <br>
        <button type="submit" class="btn btn-success" onclick="return confirm('Are you sure? You cannot change your vote.')">✅ Submit Vote</button>
        <a href="{% url 'vote_list' %}" class="btn" style="background:#e2e8f0; margin-left:0.5rem;">Cancel</a>
    </form>
</div>
{% endblock %}
```

---

### voting/templates/voting/results.html

```html
{% extends 'voting/base.html' %}
{% block title %}Results - {{ election.title }}{% endblock %}
{% block content %}
<div class="card">
    <h2>📊 Results: {{ election.title }}</h2>
    <p style="color:#718096; margin-bottom:1.5rem;">Total votes cast: <strong>{{ total_votes }}</strong></p>

    {% for item in candidate_data %}
    <div style="margin-bottom:1.2rem;">
        <div style="display:flex; justify-content:space-between; margin-bottom:0.3rem;">
            <span><strong>{{ forloop.counter }}. {{ item.candidate.name }}</strong>
                {% if item.candidate.party %}<span style="color:#718096;"> — {{ item.candidate.party }}</span>{% endif %}
            </span>
            <span><strong>{{ item.count }}</strong> votes ({{ item.percentage }}%)</span>
        </div>
        <div style="background:#e2e8f0; border-radius:99px; height:20px;">
            <div style="background: {% if forloop.first %}#276749{% else %}#2b6cb0{% endif %}; width:{{ item.percentage }}%; height:100%; border-radius:99px; transition:width 1s;"></div>
        </div>
    </div>
    {% empty %}
    <p>No votes have been cast yet.</p>
    {% endfor %}

    {% if candidate_data %}
    <div style="margin-top:2rem; padding:1rem; background:#f0fff4; border-radius:8px; border-left:4px solid #276749;">
        🏆 <strong>Winner: {{ candidate_data.0.candidate.name }}</strong>
        {% if candidate_data.0.candidate.party %} ({{ candidate_data.0.candidate.party }}){% endif %}
        with {{ candidate_data.0.count }} votes ({{ candidate_data.0.percentage }}%)
    </div>
    {% endif %}
</div>
<a href="{% url 'vote_list' %}" class="btn btn-primary">← Back to Elections</a>
{% endblock %}
```

---

## 10. Run the Project

```bash
python manage.py makemigrations
python manage.py migrate
python manage.py createsuperuser   # Create admin account
python manage.py runserver
```

## Admin Panel
Go to: http://127.0.0.1:8000/admin/
- Create Elections with start/end dates
- Add Candidates to each election
- Toggle elections active/inactive
- View all votes cast

## User Flow
1. Register at /register/
2. Login at /login/
3. View & vote at /vote/
4. See results at /results/<id>/
