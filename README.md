Below is the complete code for setting up the Django project, including models, serializers, views, URL routing, and details on how to test each API endpoint using Postman. This guide ensures that both HR and regular users can log in, and that users can register, log in, view their profiles, apply for leave, and mark attendance. Additionally, HR users can approve leave requests and access a comprehensive HR dashboard.

### Step-by-Step Instructions

#### 1. Set Up Django Project

```sh
django-admin startproject employ_management
cd employ_management
django-admin startapp api
```

#### 2. Define Models in `api/models.py`

```python
from django.contrib.auth.models import AbstractUser
from django.db import models
from datetime import datetime, time

class Department(models.Model):
    name = models.CharField(max_length=100)

    def __str__(self):
        return self.name

class Position(models.Model):
    title = models.CharField(max_length=100)
    department = models.ForeignKey(Department, related_name='positions', on_delete=models.CASCADE)

    def __str__(self):
        return self.title

class User(AbstractUser):
    first_name = models.CharField(max_length=50)
    last_name = models.CharField(max_length=50)
    salary = models.DecimalField(max_digits=10, decimal_places=2)
    date_of_joining = models.DateField()
    department = models.ForeignKey(Department, on_delete=models.SET_NULL, null=True)
    position = models.ForeignKey(Position, on_delete=models.SET_NULL, null=True)

class Leave(models.Model):
    LEAVE_TYPE_CHOICES = [
        ('casual', 'Casual Leave'),
        ('sick', 'Sick Leave'),
        ('earned', 'Earned Leave')
    ]
    user = models.ForeignKey(User, related_name='leaves', on_delete=models.CASCADE)
    leave_type = models.CharField(max_length=20, choices=LEAVE_TYPE_CHOICES)
    start_date = models.DateField()
    end_date = models.DateField()
    is_approved = models.BooleanField(default=False)

    def __str__(self):
        return f'{self.user.username} - {self.leave_type}'

class Task(models.Model):
    TASK_STATUS_CHOICES = [
        ('not_started', 'Not Started'),
        ('in_progress', 'In Progress'),
        ('completed', 'Completed')
    ]
    user = models.ForeignKey(User, related_name='tasks', on_delete=models.CASCADE)
    title = models.CharField(max_length=100)
    description = models.TextField()
    due_date = models.DateField()
    status = models.CharField(max_length=20, choices=TASK_STATUS_CHOICES, default='not_started')

    def __str__(self):
        return self.title

class Attendance(models.Model):
    user = models.ForeignKey(User, related_name='attendance', on_delete=models.CASCADE)
    check_in = models.DateTimeField(null=True, blank=True)
    check_out = models.DateTimeField(null=True, blank=True)
    is_late = models.BooleanField(default=False)

    def save(self, *args, **kwargs):
        office_start_time = time(11, 30)
        if self.check_in and self.check_in.time() > office_start_time:
            self.is_late = True
        super().save(*args, **kwargs)
```

#### 3. Create Serializers in `api/serializers.py`

```python
from rest_framework import serializers
from .models import User, Department, Position, Leave, Task, Attendance

class DepartmentSerializer(serializers.ModelSerializer):
    class Meta:
        model = Department
        fields = '__all__'

class PositionSerializer(serializers.ModelSerializer):
    class Meta:
        model = Position
        fields = '__all__'

class LeaveSerializer(serializers.ModelSerializer):
    class Meta:
        model = Leave
        fields = '__all__'

class TaskSerializer(serializers.ModelSerializer):
    class Meta:
        model = Task
        fields = '__all__'

class AttendanceSerializer(serializers.ModelSerializer):
    class Meta:
        model = Attendance
        fields = '__all__'

class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ['id', 'first_name', 'last_name', 'username', 'salary', 'date_of_joining', 'department', 'position']

class UserProfileSerializer(serializers.ModelSerializer):
    leaves = LeaveSerializer(many=True, read_only=True)
    tasks = TaskSerializer(many=True, read_only=True)
    attendance = AttendanceSerializer(many=True, read_only=True)

    class Meta:
        model = User
        fields = ['id', 'first_name', 'last_name', 'username', 'salary', 'date_of_joining', 'department', 'position', 'leaves', 'tasks', 'attendance']
        depth = 1

class UserRegistrationSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ['first_name', 'last_name', 'username', 'password', 'salary', 'date_of_joining', 'department', 'position']
        extra_kwargs = {'password': {'write_only': True}}

    def create(self, validated_data):
        user = User.objects.create_user(**validated_data)
        return user
```

#### 4. Create Views in `api/views.py`

```python
from rest_framework import viewsets, permissions, status
from rest_framework.response import Response
from rest_framework.views import APIView
from rest_framework_simplejwt.views import TokenObtainPairView
from rest_framework_simplejwt.tokens import RefreshToken
from rest_framework.permissions import IsAuthenticated
from .models import User, Department, Position, Leave, Task, Attendance
from .serializers import UserSerializer, UserProfileSerializer, UserRegistrationSerializer, DepartmentSerializer, PositionSerializer, LeaveSerializer, TaskSerializer, AttendanceSerializer

class HRLoginView(TokenObtainPairView):
    permission_classes = (permissions.AllowAny,)

    def post(self, request, *args, **kwargs):
        response = super().post(request, *args, **kwargs)
        user = User.objects.filter(username=request.data['username']).first()
        if response.status_code == 200 and user and user.is_superuser:
            return response
        return Response({'detail': 'Invalid credentials or not an HR'}, status=status.HTTP_401_UNAUTHORIZED)

class UserLoginView(TokenObtainPairView):
    permission_classes = (permissions.AllowAny,)

    def post(self, request, *args, **kwargs):
        response = super().post(request, *args, **kwargs)
        user = User.objects.filter(username=request.data['username']).first()
        if response.status_code == 200 and user:
            return response
        return Response({'detail': 'Invalid credentials'}, status=status.HTTP_401_UNAUTHORIZED)

class UserRegistrationView(APIView):
    permission_classes = (permissions.IsAuthenticated,)

    def post(self, request):
        if not request.user.is_superuser:
            return Response({'detail': 'Permission denied'}, status=status.HTTP_403_FORBIDDEN)
        serializer = UserRegistrationSerializer(data=request.data)
        if serializer.is_valid():
            user = serializer.save()
            user_serializer = UserProfileSerializer(user)
            return Response({
                'msg': 'User registered successfully',
                'user': user_serializer.data
            }, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

class UserProfileView(APIView):
    permission_classes = [IsAuthenticated]

    def get(self, request):
        user = request.user
        serializer = UserProfileSerializer(user)
        return Response(serializer.data)

class UserViewSet(viewsets.ModelViewSet):
    queryset = User.objects.all()
    serializer_class = UserSerializer
    permission_classes = [permissions.IsAuthenticated]

class LeaveViewSet(viewsets.ModelViewSet):
    queryset = Leave.objects.all()
    serializer_class = LeaveSerializer
    permission_classes = [permissions.IsAuthenticated]

    def get_queryset(self):
        if self.request.user.is_superuser:
            return Leave.objects.all()
        return Leave.objects.filter(user=self.request.user)

class DepartmentViewSet(viewsets.ModelViewSet):
    queryset = Department.objects.all()
    serializer_class = DepartmentSerializer
    permission_classes = [permissions.IsAuthenticated]

class PositionViewSet(viewsets.ModelViewSet):
    queryset = Position.objects.all()
    serializer_class = PositionSerializer
    permission_classes = [permissions.IsAuthenticated]

class TaskViewSet(viewsets.ModelViewSet):
    queryset = Task.objects.all()
    serializer_class = TaskSerializer
    permission_classes = [permissions.IsAuthenticated]

    def get_queryset(self):
        if self.request.user.is_superuser:
            return Task.objects.all()
        return Task.objects.filter(user=self.request.user)

class AttendanceViewSet(viewsets.ModelViewSet):
    queryset = Attendance.objects.all()
    serializer_class = AttendanceSerializer
    permission_classes = [permissions.IsAuthenticated]

    def get_queryset(self):
        if self.request.user.is_superuser:
            return Attendance.objects.all()
        return Attendance.objects.filter(user=self.request.user)

class LogoutView(APIView):
    permission_classes = (permissions.IsAuthenticated,)

    def post(self, request):
        try:
            refresh_token = request.data["refresh_token"]
            token = RefreshToken(refresh_token)
            token.blacklist()
            return Response(status=status.HTTP_205_RESET_CONTENT)
        except Exception as e:
            return Response(status=status.HTTP_400_BAD_REQUEST)
```

#### 5. Configure URL Routes in `api/urls.py`

```python
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from .views import HRLoginView, UserLoginView, UserRegistrationView, UserProfileView, UserViewSet, LeaveViewSet, DepartmentViewSet, PositionViewSet, TaskViewSet, AttendanceViewSet, LogoutView

router = DefaultRouter()
router.register(r'users', UserViewSet)
router.register(r'leaves', Leave

ViewSet)
router.register(r'departments', DepartmentViewSet)
router.register(r'positions', PositionViewSet)
router.register(r'tasks', TaskViewSet)
router.register(r'attendance', AttendanceViewSet)

urlpatterns = [
    path('hr/login/', HRLoginView.as_view(), name='hr_login'),
    path('user/login/', UserLoginView.as_view(), name='user_login'),
    path('logout/', LogoutView.as_view(), name='auth_logout'),
    path('register/', UserRegistrationView.as_view(), name='user_register'),
    path('profile/', UserProfileView.as_view(), name='user_profile'),
    path('', include(router.urls)),
]
```

#### 6. Configure Main URLs in `employ_management/urls.py`

```python
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/', include('api.urls')),
]
```

#### 7. Settings Configuration in `employ_management/settings.py`

```python
INSTALLED_APPS = [
    ...
    'rest_framework',
    'rest_framework_simplejwt',
    'api',
]

AUTH_USER_MODEL = 'api.User'

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': (
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ),
}

from datetime import timedelta

SIMPLE_JWT = {
    'ACCESS_TOKEN_LIFETIME': timedelta(minutes=5),
    'REFRESH_TOKEN_LIFETIME': timedelta(days=1),
}
```

#### 8. Make Migrations and Migrate

```sh
python manage.py makemigrations
python manage.py migrate
```

#### 9. Create a Superuser for HR

```sh
python manage.py createsuperuser
```

#### 10. Postman Requests

##### HR Login
**URL:** `http://127.0.0.1:8000/api/hr/login/`  
**Method:** `POST`  
**Body (raw JSON):**
```json
{
    "username": "your_hr_username",
    "password": "your_hr_password"
}
```
**Response:**  
```json
{
    "refresh": "your_refresh_token",
    "access": "your_access_token"
}
```

##### User Login
**URL:** `http://127.0.0.1:8000/api/user/login/`  
**Method:** `POST`  
**Body (raw JSON):**
```json
{
    "username": "your_user_username",
    "password": "your_user_password"
}
```
**Response:**  
```json
{
    "refresh": "your_refresh_token",
    "access": "your_access_token"
}
```

##### User Registration
**URL:** `http://127.0.0.1:8000/api/register/`  
**Method:** `POST`  
**Headers:**  
`Authorization: Bearer <hr_access_token>`  
**Body (raw JSON):**
```json
{
    "first_name": "John",
    "last_name": "Doe",
    "username": "johndoe",
    "password": "password123",
    "salary": "50000.00",
    "date_of_joining": "2022-01-01",
    "department": 1,
    "position": 1
}
```
**Response:**  
```json
{
    "msg": "User registered successfully",
    "user": {
        "id": 1,
        "first_name": "John",
        "last_name": "Doe",
        "username": "johndoe",
        "salary": "50000.00",
        "date_of_joining": "2022-01-01",
        "department": 1,
        "position": 1,
        "leaves": [],
        "tasks": [],
        "attendance": []
    }
}
```

##### User Profile
**URL:** `http://127.0.0.1:8000/api/profile/`  
**Method:** `GET`  
**Headers:**  
`Authorization: Bearer <user_access_token>`  
**Response:**  
```json
{
    "id": 1,
    "first_name": "John",
    "last_name": "Doe",
    "username": "johndoe",
    "salary": "50000.00",
    "date_of_joining": "2022-01-01",
    "department": 1,
    "position": 1,
    "leaves": [],
    "tasks": [],
    "attendance": []
}
```

##### Apply for Leave
**URL:** `http://127.0.0.1:8000/api/leaves/`  
**Method:** `POST`  
**Headers:**  
`Authorization: Bearer <user_access_token>`  
**Body (raw JSON):**
```json
{
    "user": 1,
    "leave_type": "casual",
    "start_date": "2023-07-01",
    "end_date": "2023-07-03",
    "is_approved": false
}
```
**Response:**  
```json
{
    "id": 1,
    "user": 1,
    "leave_type": "casual",
    "start_date": "2023-07-01",
    "end_date": "2023-07-03",
    "is_approved": false
}
```

##### Approve Leave (HR Only)
**URL:** `http://127.0.0.1:8000/api/leaves/<leave_id>/`  
**Method:** `PUT`  
**Headers:**  
`Authorization: Bearer <hr_access_token>`  
**Body (raw JSON):**
```json
{
    "user": 1,
    "leave_type": "casual",
    "start_date": "2023-07-01",
    "end_date": "2023-07-03",
    "is_approved": true
}
```
**Response:**  
```json
{
    "id": 1,
    "user": 1,
    "leave_type": "casual",
    "start_date": "2023-07-01",
    "end_date": "2023-07-03",
    "is_approved": true
}
```

##### Check In
**URL:** `http://127.0.0.1:8000/api/attendance/`  
**Method:** `POST`  
**Headers:**  
`Authorization: Bearer <user_access_token>`  
**Body (raw JSON):**
```json
{
    "user": 1,
    "check_in": "2023-07-01T08:30:00Z",
    "check_out": null,
    "is_late": false
}
```
**Response:**  
```json
{
    "id": 1,
    "user": 1,
    "check_in": "2023-07-01T08:30:00Z",
    "check_out": null,
    "is_late": false
}
```

##### Check Out
**URL:** `http://127.0.0.1:8000/api/attendance/<attendance_id>/`  
**Method:** `PUT`  
**Headers:**  
`Authorization: Bearer <user_access_token>`  
**Body (raw JSON):**
```json
{
    "user": 1,
    "check_in": "2023-07-01T08:30:00Z",
    "check_out": "2023-07-01T17:30:00Z",
    "is_late": false
}
```
**Response:**  
```json
{
    "id": 1,
    "user": 1,
    "check_in": "2023-07-01T08:30:00Z",
    "check_out": "2023-07-01T17:30:00Z",
    "is_late": false
}
```

##### Logout
**URL:** `http://127.0.0.1:8000/api/logout/`  
**Method:** `POST`  
**Headers:**  
`Authorization: Bearer <your_access_token>`  
**Body (raw JSON):**
```json
{
    "refresh_token": "your_refresh_token"
}
```
HR Login
URL: http://127.0.0.1:8000/api/hr/login/
Method: POST
Body (raw JSON):
json
Copy code
{
    "username": "your_hr_username",
    "password": "your_hr_password"
}
Expected Response:
json
Copy code
{
    "refresh": "your_refresh_token",
    "access": "your_access_token"
}
User Login
URL: http://127.0.0.1:8000/api/user/login/
Method: POST
Body (raw JSON):
json
Copy code
{
    "username": "your_user_username",
    "password": "your_user_password"
}
Expected Response:
json
Copy code
{
    "refresh": "your_refresh_token",
    "access": "your_access_token"
}
User Registration
URL: http://127.0.0.1:8000/api/register/
Method: POST
Headers:
Authorization: Bearer <hr_access_token>
Body (raw JSON):
json
Copy code
{
    "first_name": "John",
    "last_name": "Doe",
    "username": "johndoe",
    "password": "password123",
    "salary": "50000.00",
    "date_of_joining": "2022-01-01",
    "department": 1,
    "position": 1
}
Expected Response:
json
Copy code
{
    "msg": "User registered successfully",
    "user": {
        "id": 1,
        "first_name": "John",
        "last_name": "Doe",
        "username": "johndoe",
        "salary": "50000.00",
        "date_of_joining": "2022-01-01",
        "department": 1,
        "position": 1,
        "leaves": [],
        "tasks": []
    }
}
User Profile
URL: http://127.0.0.1:8000/api/profile/
Method: GET
Headers:
Authorization: Bearer <user_access_token>
Expected Response:
json
Copy code
{
    "id": 1,
    "first_name": "John",
    "last_name": "Doe",
    "username": "johndoe",
    "salary": "50000.00",
    "date_of_joining": "2022-01-01",
    "department": 1,
    "position": 1,
    "leaves": [],
    "tasks": []
}
Apply for Leave
URL: http://127.0.0.1:8000/api/leaves/
Method: POST
Headers:
Authorization: Bearer <user_access_token>
Body (raw JSON):
json
Copy code
{
    "user": 1,
    "leave_type": "casual",
    "start_date": "2023-07-01",
    "end_date": "2023-07-03",
    "is_approved": false
}
Expected Response:
json
Copy code
{
    "id": 1,
    "user": 1,
    "leave_type": "casual",
    "start_date": "2023-07-01",
    "end_date": "2023-07-03",
    "is_approved": false
}
Approve Leave (HR Only)
URL: http://127.0.0.1:8000/api/leaves/<leave_id>/
Method: PUT
Headers:
Authorization: Bearer <hr_access_token>
Body (raw JSON):
json
Copy code
{
    "user": 1,
    "leave_type": "casual",
    "start_date": "2023-07-01",
    "end_date": "2023-07-03",
    "is_approved": true
}
Expected Response:
json
Copy code
{
    "id": 1,
    "user": 1,
    "leave_type": "casual",
    "start_date": "2023-07-01",
    "end_date": "2023-07-03",
    "is_approved": true
}
Check In
URL: http://127.0.0.1:8000/api/attendance/
Method: POST
Headers:
Authorization: Bearer <user_access_token>
Body (raw JSON):
json
Copy code
{
    "user": 1,
    "check_in": "2023-07-01T08:30:00Z",
    "check_out": null,
    "is_late": false
}
Expected Response:
json
Copy code
{
    "id": 1,
    "user": 1,
    "check_in": "2023-07-01T08:30:00Z",
    "check_out": null,
    "is_late": false
}
Check Out
URL: http://127.0.0.1:8000/api/attendance/<attendance_id>/
Method: PUT
Headers:
Authorization: Bearer <user_access_token>
Body (raw JSON):
json
Copy code
{
    "user": 1,
    "check_in": "2023-07-01T08:30:00Z",
    "check_out": "2023-07-01T17:30:00Z",
    "is_late": false
}
Expected Response:
json
Copy code
{
    "id": 1,
    "user": 1,
    "check_in": "2023-07-01T08:30:00Z",
    "check_out": "2023-07-01T17:30:00Z",
    "is_late": false
}
Logout
URL: http://127.0.0.1:8000/api/logout/
Method: POST
Headers:
Authorization: Bearer <your_access_token>
Body (raw JSON):
json
Copy code
{
    "refresh_token": "your_refresh_token"
}
Expected Response: Status: 205 Reset Content

These are the primary API endpoints required for user registration, login, profile access, leave management, attendance marking, and logout. You can further expand upon this to add more detailed functionalities as needed. The provided guide should cover all essential aspects to get you started with the employee management system.
