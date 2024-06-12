Sure, here is the complete Django code for implementing the HR dashboard that shows all user details, including pending and approved leaves, along with the ability to manage leaves, attendance, and tasks.

### 1. Set Up Django Project

```sh
django-admin startproject employ_management
cd employ_management
django-admin startapp api
```

### 2. Define Models in `api/models.py`

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
    casual_leaves_taken = models.IntegerField(default=0)
    sick_leaves_taken = models.IntegerField(default=0)
    remaining_leaves = models.IntegerField(default=20)

class Leave(models.Model):
    LEAVE_TYPE_CHOICES = [
        ('casual', 'Casual Leave'),
        ('sick', 'Sick Leave'),
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
        ('completed', 'Completed'),
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

### 3. Create Serializers in `api/serializers.py`

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
        fields = ['id', 'first_name', 'last_name', 'username', 'salary', 'date_of_joining', 'department', 'position', 'leaves', 'tasks', 'attendance', 'casual_leaves_taken', 'sick_leaves_taken', 'remaining_leaves']
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

class HRUserLeaveSerializer(serializers.ModelSerializer):
    pending_leaves = serializers.SerializerMethodField()
    approved_leaves = serializers.SerializerMethodField()

    class Meta:
        model = User
        fields = ['id', 'first_name', 'last_name', 'username', 'department', 'position', 'remaining_leaves', 'pending_leaves', 'approved_leaves']

    def get_pending_leaves(self, obj):
        pending_leaves = Leave.objects.filter(user=obj, is_approved=False).count()
        return pending_leaves

    def get_approved_leaves(self, obj):
        approved_leaves = Leave.objects.filter(user=obj, is_approved=True).count()
        return approved_leaves
```

### 4. Create Views in `api/views.py`

```python
from rest_framework import viewsets, permissions, status
from rest_framework.response import Response
from rest_framework.views import APIView
from rest_framework_simplejwt.views import TokenObtainPairView
from rest_framework.permissions import IsAuthenticated
from django.shortcuts import render
from datetime import datetime
from .models import User, Department, Position, Leave, Task, Attendance
from .serializers import UserSerializer, UserProfileSerializer, UserRegistrationSerializer, DepartmentSerializer, PositionSerializer, LeaveSerializer, TaskSerializer, AttendanceSerializer, HRUserLeaveSerializer

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

class HRUserProfileView(APIView):
    permission_classes = [IsAuthenticated]

    def get(self, request, user_id):
        if not request.user.is_superuser:
            return Response({'detail': 'Permission denied'}, status=status.HTTP_403_FORBIDDEN)
        user = User.objects.get(id=user_id)
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

    def create(self, request, *args, **kwargs):
        user = request.user
        leave_type = request.data.get('leave_type')
        start_date = request.data.get('start_date')
        end_date = request.data.get('end_date')

        # Calculate number of leave days
        leave_days = (datetime.strptime(end_date, '%Y-%m-%d') - datetime.strptime(start_date, '%Y-%m-%d')).days + 1

        if leave_type == 'casual' and (user.casual_leaves_taken + leave_days) > 10:
            return Response({"detail": "Exceeds casual leave quota."}, status=status.HTTP_400_BAD_REQUEST)
        elif leave_type == 'sick' and (user.sick_leaves_taken + leave_days) > 10:
            return Response({"detail": "Exceeds sick leave quota."}, status=status.HTTP_400_BAD

_REQUEST)

        # Update leave taken count
        if leave_type == 'casual':
            user.casual_leaves_taken += leave_days
        elif leave_type == 'sick':
            user.sick_leaves_taken += leave_days

        user.remaining_leaves -= leave_days
        user.save()

        return super().create(request, *args, **kwargs)

class TaskViewSet(viewsets.ModelViewSet):
    queryset = Task.objects.all()
    serializer_class = TaskSerializer
    permission_classes = [permissions.IsAuthenticated]

class AttendanceViewSet(viewsets.ModelViewSet):
    queryset = Attendance.objects.all()
    serializer_class = AttendanceSerializer
    permission_classes = [permissions.IsAuthenticated]

class CheckInView(APIView):
    permission_classes = [IsAuthenticated]

    def post(self, request):
        user = request.user
        attendance = Attendance(user=user, check_in=datetime.now())
        attendance.save()
        return Response({"msg": "Checked in successfully"}, status=status.HTTP_201_CREATED)

class CheckOutView(APIView):
    permission_classes = [IsAuthenticated]

    def post(self, request):
        user = request.user
        attendance = Attendance.objects.filter(user=user, check_out__isnull=True).order_by('-check_in').first()
        if not attendance:
            return Response({"detail": "No check-in found to check out from."}, status=status.HTTP_400_BAD_REQUEST)
        attendance.check_out = datetime.now()
        attendance.save()
        return Response({"msg": "Checked out successfully"}, status=status.HTTP_200_OK)

class HRDashboardView(APIView):
    permission_classes = [IsAuthenticated]

    def get(self, request):
        if not request.user.is_superuser:
            return Response({'detail': 'Permission denied'}, status=status.HTTP_403_FORBIDDEN)
        users = User.objects.all()
        serializer = HRUserLeaveSerializer(users, many=True)
        return Response(serializer.data)

def hr_dashboard(request):
    return render(request, 'hr_dashboard.html')
```

### 5. Create URL Routes in `api/urls.py`

```python
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from .views import HRLoginView, UserLoginView, UserRegistrationView, UserProfileView, HRUserProfileView, UserViewSet, LeaveViewSet, TaskViewSet, AttendanceViewSet, CheckInView, CheckOutView, HRDashboardView, hr_dashboard

router = DefaultRouter()
router.register(r'users', UserViewSet)
router.register(r'leaves', LeaveViewSet)
router.register(r'tasks', TaskViewSet)
router.register(r'attendance', AttendanceViewSet)

urlpatterns = [
    path('hr/login/', HRLoginView.as_view(), name='hr_login'),
    path('user/login/', UserLoginView.as_view(), name='user_login'),
    path('register/', UserRegistrationView.as_view(), name='user_register'),
    path('profile/', UserProfileView.as_view(), name='user_profile'),
    path('hr/user/<int:user_id>/', HRUserProfileView.as_view(), name='hr_user_profile'),
    path('hr/dashboard/data/', HRDashboardView.as_view(), name='hr_dashboard_data'),
    path('hr/dashboard/', hr_dashboard, name='hr_dashboard'),
    path('checkin/', CheckInView.as_view(), name='checkin'),
    path('checkout/', CheckOutView.as_view(), name='checkout'),
    path('', include(router.urls)),
]
```

### 6. Create Templates

Create the `hr_dashboard.html` template in `api/templates/`.

`api/templates/hr_dashboard.html`:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>HR Dashboard</title>
    <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/4.0.0/css/bootstrap.min.css">
</head>
<body>
    <div class="container mt-5">
        <h1>HR Dashboard</h1>
        <table class="table table-striped">
            <thead>
                <tr>
                    <th>User ID</th>
                    <th>First Name</th>
                    <th>Last Name</th>
                    <th>Username</th>
                    <th>Department</th>
                    <th>Position</th>
                    <th>Remaining Leaves</th>
                    <th>Pending Leaves</th>
                    <th>Approved Leaves</th>
                </tr>
            </thead>
            <tbody id="user-data">
                <!-- Data will be populated here -->
            </tbody>
        </table>
    </div>

    <script>
        document.addEventListener("DOMContentLoaded", function () {
            fetch('/api/hr/dashboard/data/')
                .then(response => response.json())
                .then(data => {
                    const tableBody = document.getElementById("user-data");
                    data.forEach(user => {
                        const row = document.createElement("tr");
                        row.innerHTML = `
                            <td>${user.id}</td>
                            <td>${user.first_name}</td>
                            <td>${user.last_name}</td>
                            <td>${user.username}</td>
                            <td>${user.department ? user.department.name : ''}</td>
                            <td>${user.position ? user.position.title : ''}</td>
                            <td>${user.remaining_leaves}</td>
                            <td>${user.pending_leaves}</td>
                            <td>${user.approved_leaves}</td>
                        `;
                        tableBody.appendChild(row);
                    });
                })
                .catch(error => console.error('Error fetching user data:', error));
        });
    </script>
</body>
</html>
```

### 7. Update Django Settings

In `employ_management/settings.py`, add the necessary configurations.

```python
# Add 'api' to INSTALLED_APPS
INSTALLED_APPS = [
    ...
    'rest_framework',
    'rest_framework_simplejwt',
    'api',
]

# Add custom user model
AUTH_USER_MODEL = 'api.User'

# Configure REST framework and Simple JWT
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ],
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
}

# Add Django templates
TEMPLATES = [
    ...
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [BASE_DIR / 'templates'],
        ...
        'OPTIONS': {
            ...
            'context_processors': [
                ...
                'django.template.context_processors.request',
            ],
        },
    },
]
```

### 8. Apply Migrations and Run Server

```sh
python manage.py makemigrations
python manage.py migrate
python manage.py createsuperuser
python manage.py runserver
```

### Summary

This complete Django code sets up an HR management system with functionalities to view all user details, manage leaves, attendance, tasks, and provides a dashboard for HR to see user details, including pending and approved leaves. The system ensures security by allowing only authenticated users and specific permissions for HR and regular users.
