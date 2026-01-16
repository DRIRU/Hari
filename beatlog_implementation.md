# Device Health Monitoring System - Complete Documentation

---

## 1. User Hierarchy

```
superAdmin → clientAdmin → zoneAdmin → stateAdmin → districtAdmin → locationOperator
                                                                         └── Site → Device → DVR → Cameras/HDDs
```

---

## 2. Timing Configuration

| Component | Interval | Purpose |
|-----------|----------|---------|
| Device heartbeat | **60s** | Sends health data |
| Offline threshold | **120s** | Mark device offline |
| Cloud segment | **5min** | Video segment length |
| Cloud offline threshold | **10min** | Cloud recording down |
| Location summary | **5min** | Aggregate stats |
| Hierarchy rollup | **15min** | District→State→Zone |
| Daily summary | **Midnight** | Generate daily stats |

---

## 3. Beat Log Payload

```json
{
  "api_key": "xxx-xxx-xxx",
  "device_id": "DEVICE-001",
  "device_name": "Main Server",
  "device_status": "ONLINE",
  "device_ip": "192.168.1.100",
  "location_id": "LOC-001",
  "cpu_percent": 45.2,
  "ram_percent": 62.8,
  "uptime_seconds": 864000,
  "dvr_details": [
    {
      "dvr_id": "DVR-001",
      "dvr_name": "Main Building",
      "status": "ONLINE",
      "model": "Hikvision DS-7608",
      "is_recording": true,
      "hdds": [
        {"hdd_id": "HDD-1", "name": "Seagate 2TB", "total_gb": 2000, "used_gb": 1500, "percent": 75, "status": "OK", "temperature_c": 38, "recording_days": 14}
      ],
      "cameras": [
        {"camera_id": "CAM-001", "name": "Entrance", "channel": 1, "status": "UP", "resolution": "1920x1080", "fps": 25,
         "local_recording": {"is_recording": true, "recording_days": 14},
         "cloud_recording": {"is_active": true, "last_upload": "2026-01-17T01:00:00"}}
      ]
    }
  ]
}
```

---

## 4. All Models

### stream_Device (Modify)
```python
last_heartbeat = models.DateTimeField(null=True)
is_online = models.BooleanField(default=True)
location_id = models.CharField(max_length=100, null=True, db_index=True)
```

### DeviceHealthLog
```python
class DeviceHealthLog(models.Model):
    device = models.ForeignKey('stream_Device', on_delete=models.CASCADE)
    user = models.ForeignKey('CustomUser', on_delete=models.CASCADE)
    device_id = models.CharField(max_length=100, db_index=True)
    location_id = models.CharField(max_length=100, db_index=True)
    device_status = models.CharField(max_length=20)
    device_ip = models.CharField(max_length=50, null=True)
    cpu_percent = models.FloatField(null=True)
    ram_percent = models.FloatField(null=True)
    uptime_seconds = models.BigIntegerField(null=True)
    dvr_details = models.JSONField(default=list)
    created_at = models.DateTimeField(auto_now_add=True, db_index=True)
```

### DailyHealthSummary
```python
class DailyHealthSummary(models.Model):
    device = models.ForeignKey('stream_Device', on_delete=models.CASCADE)
    date = models.DateField(db_index=True)
    uptime_percent = models.FloatField()
    downtime_minutes = models.IntegerField()
    offline_incidents = models.IntegerField()
    avg_cpu_percent = models.FloatField()
    avg_hdd_percent = models.FloatField()
    max_hdd_percent = models.FloatField()
    cameras_avg_online = models.FloatField()
    class Meta:
        unique_together = ['device', 'date']
```

### LocationHealthSummary
```python
class LocationHealthSummary(models.Model):
    location_id = models.CharField(max_length=100, unique=True)
    district_id = models.CharField(max_length=100, db_index=True)
    devices_total = models.IntegerField(default=0)
    devices_online = models.IntegerField(default=0)
    dvrs_total = models.IntegerField(default=0)
    dvrs_online = models.IntegerField(default=0)
    cameras_total = models.IntegerField(default=0)
    cameras_online = models.IntegerField(default=0)
    hdds_total = models.IntegerField(default=0)
    hdds_critical = models.IntegerField(default=0)
    avg_hdd_percent = models.FloatField(default=0)
    updated_at = models.DateTimeField(auto_now=True)
```

### DistrictHealthSummary
```python
class DistrictHealthSummary(models.Model):
    district_id = models.CharField(max_length=100, unique=True)
    state_id = models.CharField(max_length=100, db_index=True)
    locations_total = models.IntegerField(default=0)
    devices_total = models.IntegerField(default=0)
    devices_online = models.IntegerField(default=0)
    dvrs_total = models.IntegerField(default=0)
    dvrs_online = models.IntegerField(default=0)
    cameras_total = models.IntegerField(default=0)
    cameras_online = models.IntegerField(default=0)
    hdds_critical = models.IntegerField(default=0)
    avg_hdd_percent = models.FloatField(default=0)
    updated_at = models.DateTimeField(auto_now=True)
```

### StateHealthSummary
```python
class StateHealthSummary(models.Model):
    state_id = models.CharField(max_length=100, unique=True)
    zone_id = models.CharField(max_length=100, db_index=True)
    districts_total = models.IntegerField(default=0)
    locations_total = models.IntegerField(default=0)
    dvrs_online = models.IntegerField(default=0)
    cameras_total = models.IntegerField(default=0)
    cameras_online = models.IntegerField(default=0)
    hdds_critical = models.IntegerField(default=0)
    avg_hdd_percent = models.FloatField(default=0)
    updated_at = models.DateTimeField(auto_now=True)
```

### ZoneHealthSummary
```python
class ZoneHealthSummary(models.Model):
    zone_id = models.CharField(max_length=100, unique=True)
    states_total = models.IntegerField(default=0)
    locations_total = models.IntegerField(default=0)
    dvrs_online = models.IntegerField(default=0)
    cameras_total = models.IntegerField(default=0)
    cameras_online = models.IntegerField(default=0)
    hdds_critical = models.IntegerField(default=0)
    avg_hdd_percent = models.FloatField(default=0)
    updated_at = models.DateTimeField(auto_now=True)
```

---

## 5. API Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/health/log/` | POST | Receive heartbeat |
| `/api/health/zones/` | GET | All zones |
| `/api/health/zone/{id}/states/` | GET | Drill to states |
| `/api/health/state/{id}/districts/` | GET | Drill to districts |
| `/api/health/district/{id}/locations/` | GET | Drill to locations |
| `/api/health/location/{id}/devices/` | GET | Drill to devices |
| `/api/health/device/{id}/` | GET | Full details |
| `/api/health/device/{id}/history/?date=YYYY-MM-DD` | GET | Single day |
| `/api/health/device/{id}/history/?start=X&end=Y` | GET | Date range |

---

## 6. Historical Query Logic

| Date Range | Source | Detail |
|------------|--------|--------|
| Within 30 days | `DeviceHealthLog` | Raw (60s granular) |
| Older than 30 days | `DailyHealthSummary` | Daily aggregates |
| Custom range | Both combined | Auto-merge |

---

## 7. Celery Tasks

| Task | Schedule | Action |
|------|----------|--------|
| `check_offline_devices` | 60s | Find stale heartbeats |
| `check_cloud_recording` | 60s | Find camera upload gaps |
| `update_location_summaries` | 5min | Aggregate locations |
| `update_hierarchy_summaries` | 15min | Rollup to zones |
| `generate_daily_summaries` | Midnight | Create daily stats |
| `cleanup_old_logs` | Daily | Delete logs > 30 days |

---

## 8. Data Retention

| Data | Keep For |
|------|----------|
| `DeviceHealthLog` | 30 days |
| `DailyHealthSummary` | 1 year |
| Summary tables | Always |
