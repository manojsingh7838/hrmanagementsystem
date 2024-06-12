Sure, here's the complete rewritten code including the HR dashboard and user profile page with the additional feature of automatically filling in the check-in and check-out times for users:

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
        ('pending', 'Pending Leave'),
        ('approved', 'Approved Leave')
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
    start_date = models.DateField()
    end_date = models.DateField()
    status = models.CharField(max_length=20, choices=TASK_STATUS_CHOICES, default='not_started')
    progress = models.IntegerField(default=0)  # Progress as a percentage

    def __str__(self):
        return self.title

class Attendance(models.Model):
    user = models.ForeignKey(User, related_name='attendance', on_delete=models.CASCADE)
    check_in = models.DateTimeField(auto_now_add=True)
    check_out = models.DateTimeField(null=True, blank=True)
    is_late = models.BooleanField(default=False)

    def __str__(self):
        return f'{self.user.username} - {self.check_in}'
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

class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ['id', 'first_name', 'last_name', 'username', 'salary', 'date_of_joining', 'department', 'position']

class UserProfileSerializer(serializers.ModelSerializer):
    leaves = LeaveSerializer(many=True, read_only=True)
    tasks = TaskSerializer(many=True, read_only=True)
    attendance = serializers.PrimaryKeyRelatedField(many=True, read_only=True)

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

class AttendanceSerializer(serializers.ModelSerializer):
    class Meta:
        model = Attendance
        fields = ['user', 'check_in', 'check_out', 'is_late']
```

#### 4. Create Views in `api/views.py`

```python
from rest_framework import viewsets, permissions, status
from rest_framework.response import Response
from rest_framework.views import APIView
from rest_framework_simplejwt.views import TokenObtainPairView
from rest_framework.permissions import IsAuthenticated
from .models import User, Department, Position, Leave, Task, Attendance
from .serializers import UserSerializer, UserProfileSerializer, UserRegistrationSerializer, DepartmentSerializer, PositionSerializer, LeaveSerializer, TaskSerializer, AttendanceSerializer
from datetime import datetime

class HRLoginView(TokenObtainPairView):
    permission_classes = (permissions.AllowAny,)

    def post(self, request

, *args, **kwargs):
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

class AttendanceViewSet(viewsets.ModelViewSet):
    queryset = Attendance.objects.all()
    serializer_class = AttendanceSerializer
    permission_classes = [permissions.IsAuthenticated]

    def create(self, request, *args, **kwargs):
        user = request.user
        # Check if the user has already checked in for today
        today = datetime.now().date()
        existing_attendance = Attendance.objects.filter(user=user, check_in__date=today).first()
        if existing_attendance:
            return Response({"detail": "User has already checked in for today."}, status=status.HTTP_400_BAD_REQUEST)

        check_in_time = datetime.now()
        attendance = Attendance.objects.create(user=user, check_in=check_in_time)
        serializer = self.get_serializer(attendance)
        return Response(serializer.data, status=status.HTTP_201_CREATED)

    def update(self, request, *args, **kwargs):
        user = request.user
        today = datetime.now().date()
        attendance = Attendance.objects.filter(user=user, check_in__date=today).first()
        if not attendance:
            return Response({"detail": "User has not checked in today."}, status=status.HTTP_400_BAD_REQUEST)
        if attendance.check_out:
            return Response({"detail": "User has already checked out."}, status=status.HTTP_400_BAD_REQUEST)

        check_out_time = datetime.now()
        attendance.check_out = check_out_time
        attendance.save()
        serializer = self.get_serializer(attendance)
        return Response(serializer.data)
```

#### 5. Configure URL Routes in `api/urls.py`

```python
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from .views import HRLoginView, UserLoginView, UserRegistrationView, UserProfileView, UserViewSet, LeaveViewSet, DepartmentViewSet, PositionViewSet, TaskViewSet, AttendanceViewSet

router = DefaultRouter()
router.register(r'users', UserViewSet)
router.register(r'leaves', LeaveViewSet)
router.register(r'departments', DepartmentViewSet)
router.register(r'positions', PositionViewSet)
router.register(r'tasks', TaskViewSet)
router.register(r'attendance', AttendanceViewSet)

urlpatterns = [
    path('hr/login/', HRLoginView.as_view(), name='hr_login'),
    path('user/login/', UserLoginView.as_view(), name='user_login'),
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

This implementation ensures that when a user checks in, the check-in time is automatically recorded, and when they check out, the check-out time is recorded automatically.
