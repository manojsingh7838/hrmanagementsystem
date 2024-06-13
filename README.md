Certainly! Here's the complete implementation with the necessary updates to show real-time checking of user attendance, and an HR dashboard that displays all user information. The code includes models, serializers, views, and URLs.

### 1. Models in `api/models.py`

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

    @property
    def total_leaves_taken(self):
        return self.casual_leaves_taken + self.sick_leaves_taken

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

### 2. Serializers in `api/serializers.py`

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
        fields = ['id', 'first_name', 'last_name', 'username', 'salary', 'date_of_joining', 'department', 'position', 'leaves', 'tasks', 'attendance', 'casual_leaves_taken', 'sick_leaves_taken', 'remaining_leaves', 'total_leaves_taken']
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
        fields = ['id', 'first_name', 'last_name', 'username', 'department', 'position', 'remaining_leaves', 'pending_leaves', 'approved_leaves', 'total_leaves_taken']

    def get_pending_leaves(self, obj):
        pending_leaves = Leave.objects.filter(user=obj, is_approved=False).count()
        return pending_leaves

    def get_approved_leaves(self, obj):
        approved_leaves = Leave.objects.filter(user=obj, is_approved=True).count()
        return approved_leaves
```

### 3. Views in `api/views.py`

```python
from rest_framework import viewsets, permissions, status
from rest_framework.response import Response
from rest_framework.views import APIView
from rest_framework_simplejwt.views import TokenObtainPairView
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
    permission_classes = [permissions.IsAuthenticated]

    def get(self, request):
        user = request.user
        serializer = UserProfileSerializer(user)
        return Response(serializer.data)

class HRUserProfileView(APIView):
    permission_classes = [permissions.IsAuthenticated]

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
            return Response({"detail": "Exceeds sick leave quota."}, status=status.HTTP

_400_BAD_REQUEST)
        
        # If approved, update user's leave count
        if request.data.get('is_approved'):
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
    permission_classes = [permissions.AllowAny]

    def post(self, request, *args, **kwargs):
        user_id = request.data.get('user_id')
        user = User.objects.get(id=user_id)
        now = datetime.now()

        # Check if user has already checked in today
        if Attendance.objects.filter(user=user, check_in__date=now.date()).exists():
            return Response({"detail": "Already checked in today."}, status=status.HTTP_400_BAD_REQUEST)
        
        is_late = now.time() > datetime.strptime('09:00', '%H:%M').time()
        attendance = Attendance.objects.create(user=user, is_late=is_late)
        return Response({"detail": "Checked in successfully.", "attendance_id": attendance.id, "check_in_time": attendance.check_in}, status=status.HTTP_201_CREATED)

class CheckOutView(APIView):
    permission_classes = [permissions.AllowAny]

    def post(self, request, *args, **kwargs):
        user_id = request.data.get('user_id')
        user = User.objects.get(id=user_id)
        now = datetime.now()

        # Get today's check-in
        attendance = Attendance.objects.filter(user=user, check_in__date=now.date()).first()
        if not attendance or attendance.check_out:
            return Response({"detail": "Cannot check out."}, status=status.HTTP_400_BAD_REQUEST)
        
        attendance.check_out = now
        attendance.save()
        return Response({"detail": "Checked out successfully.", "check_out_time": attendance.check_out}, status=status.HTTP_200_OK)

class HRUserLeaveStatusView(APIView):
    permission_classes = [permissions.IsAuthenticated]

    def get(self, request):
        if not request.user.is_superuser:
            return Response({'detail': 'Permission denied'}, status=status.HTTP_403_FORBIDDEN)
        users = User.objects.all()
        serializer = HRUserLeaveSerializer(users, many=True)
        return Response(serializer.data)

class HRDashboardView(APIView):
    permission_classes = [permissions.IsAuthenticated]

    def get(self, request):
        if not request.user.is_superuser:
            return Response({'detail': 'Permission denied'}, status=status.HTTP_403_FORBIDDEN)
        users = User.objects.all()
        departments = Department.objects.all()
        positions = Position.objects.all()
        leaves = Leave.objects.all()
        tasks = Task.objects.all()
        attendance = Attendance.objects.all()

        user_serializer = UserSerializer(users, many=True)
        department_serializer = DepartmentSerializer(departments, many=True)
        position_serializer = PositionSerializer(positions, many=True)
        leave_serializer = LeaveSerializer(leaves, many=True)
        task_serializer = TaskSerializer(tasks, many=True)
        attendance_serializer = AttendanceSerializer(attendance, many=True)

        return Response({
            'users': user_serializer.data,
            'departments': department_serializer.data,
            'positions': position_serializer.data,
            'leaves': leave_serializer.data,
            'tasks': task_serializer.data,
            'attendance': attendance_serializer.data
        })
```

### 4. URLs in `api/urls.py`

```python
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from .views import (
    UserViewSet, LeaveViewSet, TaskViewSet, AttendanceViewSet, UserRegistrationView,
    UserProfileView, HRUserProfileView, CheckInView, CheckOutView, HRLoginView, UserLoginView, HRUserLeaveStatusView, HRDashboardView
)

router = DefaultRouter()
router.register(r'users', UserViewSet)
router.register(r'leaves', LeaveViewSet)
router.register(r'tasks', TaskViewSet)
router.register(r'attendance', AttendanceViewSet)

urlpatterns = [
    path('auth/hr-login/', HRLoginView.as_view(), name='hr_login'),
    path('auth/user-login/', UserLoginView.as_view(), name='user_login'),
    path('auth/register/', UserRegistrationView.as_view(), name='user_register'),
    path('profile/', UserProfileView.as_view(), name='user_profile'),
    path('profile/<int:user_id>/', HRUserProfileView.as_view(), name='hr_user_profile'),
    path('check-in/', CheckInView.as_view(), name='check_in'),
    path('check-out/', CheckOutView.as_view(), name='check_out'),
    path('hr/leaves-status/', HRUserLeaveStatusView.as_view(), name='hr_user_leaves_status'),
    path('hr/dashboard/', HRDashboardView.as_view(), name='hr_dashboard'),
    path('', include(router.urls)),
]
```

### Summary of Changes:

1. **Models**: Added the `total_leaves_taken` property to the `User` model.
2. **Serializers**: Adjusted `UserProfileSerializer` to include `total_leaves_taken` and created `HRUserLeaveSerializer` for HR-specific views.
3. **Views**: Implemented `CheckInView` and `CheckOutView` to allow check-ins and check-outs using user IDs with real-time checking. Added `HRDashboardView` to provide a comprehensive dashboard for HR.
4. **URLs**: Configured URLs for the new views.

These changes ensure that users can check in and out in real-time, HR can see all user information on a dashboard, and leave tracking is fully implemented.
