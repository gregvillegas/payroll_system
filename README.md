
# MI Core: Django Implementation Blueprint

This document outlines the core Django architecture for the MI Core Payroll & Attendance system, designed for integration with Hikvision DS-K1T671MF terminals and compliance with Philippine Labor Laws (RA 10173, etc.).

## 1. Project Structure
```text
bayani_core/
├── manage.py
├── core/                  # Project Settings & Config
├── accounts/              # User, Role & Permission Management
├── employees/             # Employee Profiles & Demographics
├── attendance/            # Hikvision Integration & DTR Workflow
├── payroll/               # Payroll Processing & Premium Pay Logic
├── compliance/            # Statutory Reports & Audit Logs
└── templates/             # HTML Templates (Referencing Stitch Designs)
```

## 2. Core Models (`models.py`)

### Employees Module
```python
from django.db import models
from django.contrib.auth.models import AbstractUser

class User(AbstractUser):
    ROLE_CHOICES = [
        ('ADMIN', 'System Administrator'),
        ('HR', 'HR Manager'),
        ('EMPLOYEE', 'Standard Employee'),
    ]
    role = models.CharField(max_length=20, choices=ROLE_CHOICES)

class Employee(models.Model):
    user = models.OneToOneField(User, on_delete=models.CASCADE)
    employee_id = models.CharField(max_length=20, unique=True)
    department = models.CharField(max_length=100)
    tin = models.CharField(max_length=15, blank=True)
    sss_no = models.CharField(max_length=15, blank=True)
    philhealth_no = models.CharField(max_length=15, blank=True)
    pagibig_no = models.CharField(max_length=15, blank=True)
    is_exempt_from_dtr = models.BooleanField(default=False)
```

### Attendance Module (Hikvision Integration)
```python
class BiometricDevice(models.Model):
    name = models.CharField(max_length=100)
    ip_address = models.GenericIPAddressField()
    serial_number = models.CharField(max_length=100, unique=True)
    status = models.BooleanField(default=True) # Online/Offline

class AttendanceLog(models.Model):
    employee = models.ForeignKey(Employee, on_delete=models.CASCADE)
    timestamp = models.DateTimeField()
    log_type = models.CharField(max_length=10, choices=[('IN', 'Time In'), ('OUT', 'Time Out')])
    device = models.ForeignKey(BiometricDevice, on_delete=models.SET_NULL, null=True)

class DTRCorrection(models.Model):
    employee = models.ForeignKey(Employee, on_delete=models.CASCADE)
    log_date = models.DateField()
    requested_time = models.TimeField()
    reason = models.TextField()
    status = models.CharField(max_length=20, default='PENDING') # APPROVED, REJECTED
```

## 3. Payroll Logic (`payroll/logic.py`)
This module handles PH-specific calculations for SSS, PhilHealth, Pag-IBIG, and Tax.

```python
def calculate_premium_pay(regular_rate, hours, type='OT'):
    # PH Labor Law Multipliers
    multipliers = {
        'OT': 1.25,
        'REST_DAY': 1.30,
        'SPECIAL_HOLIDAY': 1.30,
        'REGULAR_HOLIDAY': 2.00,
        'NIGHT_DIFF': 1.10
    }
    return regular_rate * hours * multipliers.get(type, 1.0)

def calculate_sss_contribution(gross_pay):
    # Simplified SSS Table Logic
    if gross_pay < 4250: return 180.00
    # ... dynamic table lookup ...
    return min(gross_pay * 0.045, 1350.00)
```

## 4. Data Privacy Audit Log (`compliance/middleware.py`)
Middleware to ensure all access to sensitive PII (TIN, SSS) is logged as per RA 10173.

```python
class PrivacyAuditMiddleware:
    def process_view(self, request, view_func, view_args, view_kwargs):
        if "/api/employee/sensitive-data/" in request.path:
            AuditLog.objects.create(
                user=request.user,
                action="VIEW_SENSITIVE_DATA",
                resource=request.path,
                ip_address=request.META.get('REMOTE_ADDR')
            )
```

## 5. Deployment Considerations
- **Hardware Sync:** Use a Celery task to poll the Hikvision ISAPI or SDK every 5 minutes.
- **Reporting:** Use `ReportLab` or `WeasyPrint` to generate the BIR-compliant PDF payslips seen in the designs.


