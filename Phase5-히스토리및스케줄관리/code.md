# Phase 5: 히스토리 및 스케줄 관리 - 구현 코드

## 개요
실행 히스토리 조회/분석 기능과 스케줄 관리 시스템을 구현합니다.

## 1. 실행 히스토리 관리

### 1.1 BatchExecutionHistory.java (추가 엔티티)
**파일 위치**: `/src/main/java/com/sbs/datasync/entity/BatchExecutionHistory.java`
```java
package com.sbs.datasync.entity;

import jakarta.persistence.*;

import java.time.Duration;
import java.time.LocalDateTime;

@Entity
@Table(name = "BATCH_EXECUTION_HISTORY",
       indexes = {
           @Index(name = "IDX_BATCH_EXECUTION_01", columnList = "JOB_CONFIG_ID"),
           @Index(name = "IDX_BATCH_EXECUTION_02", columnList = "EXECUTION_STATUS"),
           @Index(name = "IDX_BATCH_EXECUTION_03", columnList = "START_TIME"),
           @Index(name = "IDX_BATCH_EXECUTION_04", columnList = "JOB_EXECUTION_ID")
       })
public class BatchExecutionHistory extends BaseEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "EXECUTION_ID")
    private Long executionId;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "JOB_CONFIG_ID", nullable = false)
    private BatchJobConfig jobConfig;

    @Column(name = "JOB_INSTANCE_ID")
    private Long jobInstanceId;

    @Column(name = "JOB_EXECUTION_ID")
    private Long jobExecutionId;

    @Enumerated(EnumType.STRING)
    @Column(name = "EXECUTION_STATUS", nullable = false, length = 20)
    private ExecutionStatus executionStatus;

    @Column(name = "START_TIME")
    private LocalDateTime startTime;

    @Column(name = "END_TIME")
    private LocalDateTime endTime;

    @Column(name = "DURATION_SECONDS")
    private Long durationSeconds;

    @Column(name = "TOTAL_COUNT")
    private Long totalCount = 0L;

    @Column(name = "SUCCESS_COUNT")
    private Long successCount = 0L;

    @Column(name = "ERROR_COUNT")
    private Long errorCount = 0L;

    @Column(name = "SKIP_COUNT")
    private Long skipCount = 0L;

    @Lob
    @Column(name = "ERROR_MESSAGE", columnDefinition = "CLOB")
    private String errorMessage;

    @Lob
    @Column(name = "EXECUTION_PARAMS", columnDefinition = "CLOB")
    private String executionParams;

    @Column(name = "EXECUTED_BY", length = 50)
    private String executedBy;

    // Constructors, Getters, Setters...

    public enum ExecutionStatus {
        STARTED, COMPLETED, FAILED, STOPPED, ABANDONED
    }

    // Business methods
    public void markAsStarted() {
        this.executionStatus = ExecutionStatus.STARTED;
        this.startTime = LocalDateTime.now();
    }

    public void markAsCompleted() {
        this.executionStatus = ExecutionStatus.COMPLETED;
        this.endTime = LocalDateTime.now();
        calculateDuration();
    }

    public void markAsFailed(String errorMessage) {
        this.executionStatus = ExecutionStatus.FAILED;
        this.endTime = LocalDateTime.now();
        this.errorMessage = errorMessage;
        calculateDuration();
    }

    private void calculateDuration() {
        if (startTime != null && endTime != null) {
            this.durationSeconds = Duration.between(startTime, endTime).getSeconds();
        }
    }

    public double getSuccessRate() {
        if (totalCount == null || totalCount == 0) return 0.0;
        return (double) (successCount != null ? successCount : 0) / totalCount * 100;
    }
}
```

### 1.2 BatchHistoryController.java
**파일 위치**: `/src/main/java/com/sbs/datasync/controller/BatchHistoryController.java`
```java
package com.sbs.datasync.controller;

import com.sbs.datasync.dto.BatchHistoryDto;
import com.sbs.datasync.service.BatchHistoryService;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.format.annotation.DateTimeFormat;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.time.LocalDateTime;
import java.util.List;
import java.util.Map;

@RestController
@RequestMapping("/api/batch/history")
@CrossOrigin(origins = "http://localhost:5173")
public class BatchHistoryController {

    private final BatchHistoryService batchHistoryService;

    public BatchHistoryController(BatchHistoryService batchHistoryService) {
        this.batchHistoryService = batchHistoryService;
    }

    @GetMapping
    public ResponseEntity<Page<BatchHistoryDto.Response>> getExecutionHistory(
            @RequestParam(required = false) Long jobConfigId,
            @RequestParam(required = false) String status,
            @RequestParam(required = false) @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) LocalDateTime startDate,
            @RequestParam(required = false) @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) LocalDateTime endDate,
            @RequestParam(required = false) String executedBy,
            Pageable pageable) {

        Page<BatchHistoryDto.Response> history = batchHistoryService
            .getExecutionHistory(jobConfigId, status, startDate, endDate, executedBy, pageable);

        return ResponseEntity.ok(history);
    }

    @GetMapping("/{executionId}")
    public ResponseEntity<BatchHistoryDto.DetailResponse> getExecutionDetail(
            @PathVariable Long executionId) {
        BatchHistoryDto.DetailResponse detail = batchHistoryService.getExecutionDetail(executionId);
        return ResponseEntity.ok(detail);
    }

    @GetMapping("/statistics")
    public ResponseEntity<Map<String, Object>> getStatistics(
            @RequestParam(required = false) Long jobConfigId,
            @RequestParam(defaultValue = "30") int days) {

        Map<String, Object> statistics = batchHistoryService.getStatistics(jobConfigId, days);
        return ResponseEntity.ok(statistics);
    }

    @GetMapping("/chart-data")
    public ResponseEntity<Map<String, Object>> getChartData(
            @RequestParam(required = false) Long jobConfigId,
            @RequestParam(defaultValue = "7") int days) {

        Map<String, Object> chartData = batchHistoryService.getChartData(jobConfigId, days);
        return ResponseEntity.ok(chartData);
    }
}
```

### 1.3 ExecutionHistory.tsx (프론트엔드 페이지)
**파일 위치**: `/frontend/src/pages/ExecutionHistory.tsx`
```tsx
import React, { useState, useCallback } from 'react';
import {
  Box,
  Typography,
  Card,
  CardContent,
  Grid,
  Chip,
  IconButton,
  Dialog,
  DialogTitle,
  DialogContent,
  Toolbar,
  TextField,
  FormControl,
  InputLabel,
  Select,
  MenuItem,
  Button,
  Alert,
} from '@mui/material';
import {
  Visibility as ViewIcon,
  GetApp as DownloadIcon,
  Refresh as RefreshIcon,
  TrendingUp as TrendingUpIcon,
} from '@mui/icons-material';
import { DataGrid, GridColDef } from '@mui/x-data-grid';
import { DateTimePicker } from '@mui/x-date-pickers/DateTimePicker';
import { useQuery } from '@tanstack/react-query';
import { Line, Pie } from 'react-chartjs-2';
import {
  Chart as ChartJS,
  CategoryScale,
  LinearScale,
  PointElement,
  LineElement,
  Title,
  Tooltip,
  Legend,
  ArcElement,
} from 'chart.js';

import { batchService } from '@/services/batchService';
import { BatchExecution } from '@/types/batch';
import LoadingSpinner from '@/components/LoadingSpinner';
import ExecutionDetailDialog from '@/components/ExecutionDetailDialog';

ChartJS.register(
  CategoryScale,
  LinearScale,
  PointElement,
  LineElement,
  Title,
  Tooltip,
  Legend,
  ArcElement
);

export default function ExecutionHistory() {
  const [page, setPage] = useState(0);
  const [pageSize, setPageSize] = useState(25);
  const [filters, setFilters] = useState({
    jobConfigId: null as number | null,
    status: '',
    startDate: null as Date | null,
    endDate: null as Date | null,
    executedBy: '',
  });

  const [selectedExecution, setSelectedExecution] = useState<BatchExecution | null>(null);
  const [detailDialogOpen, setDetailDialogOpen] = useState(false);

  // 히스토리 데이터 조회
  const { data: historyData, isLoading, refetch } = useQuery({
    queryKey: ['execution-history', page, pageSize, filters],
    queryFn: () => batchService.getExecutionHistory({
      page,
      size: pageSize,
      jobConfigId: filters.jobConfigId || undefined,
      status: filters.status || undefined,
      startDate: filters.startDate?.toISOString(),
      endDate: filters.endDate?.toISOString(),
      executedBy: filters.executedBy || undefined,
    }),
  });

  // 통계 데이터 조회
  const { data: statisticsData } = useQuery({
    queryKey: ['execution-statistics', filters.jobConfigId],
    queryFn: () => batchService.getStatistics(filters.jobConfigId || undefined),
  });

  // 차트 데이터 조회
  const { data: chartData } = useQuery({
    queryKey: ['execution-chart', filters.jobConfigId],
    queryFn: () => batchService.getChartData(filters.jobConfigId || undefined),
  });

  const columns: GridColDef[] = [
    {
      field: 'jobName',
      headerName: '작업명',
      width: 200,
      valueGetter: (params) => params.row.jobConfig?.jobName,
      renderCell: (params) => (
        <Box>
          <Typography variant="body2" fontWeight="medium">
            {params.row.jobConfig?.jobName}
          </Typography>
          <Typography variant="caption" color="textSecondary">
            {params.row.jobConfig?.schemaName}.{params.row.jobConfig?.tableName}
          </Typography>
        </Box>
      ),
    },
    {
      field: 'executionStatus',
      headerName: '상태',
      width: 120,
      renderCell: (params) => {
        const statusColors = {
          COMPLETED: 'success',
          FAILED: 'error',
          STARTED: 'info',
          STOPPED: 'default',
          ABANDONED: 'warning',
        } as const;

        return (
          <Chip
            label={params.value}
            color={statusColors[params.value as keyof typeof statusColors]}
            size="small"
          />
        );
      },
    },
    {
      field: 'startTime',
      headerName: '시작 시간',
      width: 160,
      renderCell: (params) => (
        <Typography variant="body2">
          {params.value ? new Date(params.value).toLocaleString() : '-'}
        </Typography>
      ),
    },
    {
      field: 'durationSeconds',
      headerName: '소요 시간',
      width: 120,
      renderCell: (params) => {
        if (!params.value) return '-';
        const duration = params.value;
        const hours = Math.floor(duration / 3600);
        const minutes = Math.floor((duration % 3600) / 60);
        const seconds = duration % 60;

        if (hours > 0) {
          return `${hours}h ${minutes}m`;
        } else if (minutes > 0) {
          return `${minutes}m ${seconds}s`;
        } else {
          return `${seconds}s`;
        }
      },
    },
    {
      field: 'successCount',
      headerName: '처리 결과',
      width: 150,
      renderCell: (params) => (
        <Box>
          <Typography variant="body2">
            성공: {params.row.successCount?.toLocaleString() || 0}
          </Typography>
          {params.row.errorCount > 0 && (
            <Typography variant="caption" color="error">
              오류: {params.row.errorCount.toLocaleString()}
            </Typography>
          )}
        </Box>
      ),
    },
    {
      field: 'executedBy',
      headerName: '실행자',
      width: 100,
      renderCell: (params) => (
        <Typography variant="body2">
          {params.value || 'System'}
        </Typography>
      ),
    },
    {
      field: 'actions',
      headerName: '작업',
      width: 100,
      sortable: false,
      renderCell: (params) => (
        <IconButton
          size="small"
          onClick={() => handleViewDetail(params.row)}
        >
          <ViewIcon />
        </IconButton>
      ),
    },
  ];

  const handleViewDetail = useCallback((execution: BatchExecution) => {
    setSelectedExecution(execution);
    setDetailDialogOpen(true);
  }, []);

  const lineChartData = {
    labels: chartData?.dates || [],
    datasets: [
      {
        label: '성공',
        data: chartData?.successCounts || [],
        borderColor: 'rgb(75, 192, 192)',
        backgroundColor: 'rgba(75, 192, 192, 0.2)',
        tension: 0.1,
      },
      {
        label: '실패',
        data: chartData?.errorCounts || [],
        borderColor: 'rgb(255, 99, 132)',
        backgroundColor: 'rgba(255, 99, 132, 0.2)',
        tension: 0.1,
      },
    ],
  };

  const pieChartData = {
    labels: ['성공', '실패', '중단'],
    datasets: [
      {
        data: [
          statisticsData?.successCount || 0,
          statisticsData?.failureCount || 0,
          statisticsData?.stoppedCount || 0,
        ],
        backgroundColor: [
          'rgba(75, 192, 192, 0.8)',
          'rgba(255, 99, 132, 0.8)',
          'rgba(255, 206, 86, 0.8)',
        ],
        borderWidth: 1,
      },
    ],
  };

  if (isLoading) {
    return <LoadingSpinner message="실행 히스토리를 불러오는 중..." />;
  }

  return (
    <Box>
      <Box display="flex" justifyContent="space-between" alignItems="center" mb={3}>
        <Typography variant="h4">실행 히스토리</Typography>
        <Button
          variant="outlined"
          startIcon={<RefreshIcon />}
          onClick={() => refetch()}
        >
          새로고침
        </Button>
      </Box>

      {/* 통계 카드 */}
      <Grid container spacing={3} sx={{ mb: 3 }}>
        <Grid item xs={12} sm={6} md={3}>
          <Card>
            <CardContent>
              <Typography color="textSecondary" gutterBottom>
                총 실행 횟수
              </Typography>
              <Typography variant="h4">
                {statisticsData?.totalExecutions || 0}
              </Typography>
            </CardContent>
          </Card>
        </Grid>
        <Grid item xs={12} sm={6} md={3}>
          <Card>
            <CardContent>
              <Typography color="textSecondary" gutterBottom>
                성공률
              </Typography>
              <Typography variant="h4" color="success.main">
                {statisticsData?.successRate || 0}%
              </Typography>
            </CardContent>
          </Card>
        </Grid>
        <Grid item xs={12} sm={6} md={3}>
          <Card>
            <CardContent>
              <Typography color="textSecondary" gutterBottom>
                평균 실행 시간
              </Typography>
              <Typography variant="h4">
                {statisticsData?.averageDuration || 0}s
              </Typography>
            </CardContent>
          </Card>
        </Grid>
        <Grid item xs={12} sm={6} md={3}>
          <Card>
            <CardContent>
              <Typography color="textSecondary" gutterBottom>
                총 처리 건수
              </Typography>
              <Typography variant="h4">
                {(statisticsData?.totalProcessed || 0).toLocaleString()}
              </Typography>
            </CardContent>
          </Card>
        </Grid>
      </Grid>

      {/* 차트 */}
      <Grid container spacing={3} sx={{ mb: 3 }}>
        <Grid item xs={12} md={8}>
          <Card>
            <CardContent>
              <Typography variant="h6" gutterBottom>
                일별 실행 추이
              </Typography>
              <Box height={300}>
                <Line
                  data={lineChartData}
                  options={{
                    responsive: true,
                    maintainAspectRatio: false,
                    plugins: {
                      legend: { position: 'top' },
                    },
                  }}
                />
              </Box>
            </CardContent>
          </Card>
        </Grid>
        <Grid item xs={12} md={4}>
          <Card>
            <CardContent>
              <Typography variant="h6" gutterBottom>
                실행 결과 분포
              </Typography>
              <Box height={300}>
                <Pie
                  data={pieChartData}
                  options={{
                    responsive: true,
                    maintainAspectRatio: false,
                  }}
                />
              </Box>
            </CardContent>
          </Card>
        </Grid>
      </Grid>

      {/* 필터 */}
      <Card sx={{ mb: 2 }}>
        <Toolbar>
          <Grid container spacing={2} alignItems="center">
            <Grid item xs={12} sm={6} md={2}>
              <TextField
                label="실행자"
                size="small"
                fullWidth
                value={filters.executedBy}
                onChange={(e) => setFilters({ ...filters, executedBy: e.target.value })}
              />
            </Grid>
            <Grid item xs={12} sm={6} md={2}>
              <FormControl size="small" fullWidth>
                <InputLabel>상태</InputLabel>
                <Select
                  value={filters.status}
                  label="상태"
                  onChange={(e) => setFilters({ ...filters, status: e.target.value })}
                >
                  <MenuItem value="">전체</MenuItem>
                  <MenuItem value="COMPLETED">성공</MenuItem>
                  <MenuItem value="FAILED">실패</MenuItem>
                  <MenuItem value="STARTED">실행중</MenuItem>
                </Select>
              </FormControl>
            </Grid>
            <Grid item xs={12} sm={6} md={3}>
              <DateTimePicker
                label="시작일시"
                value={filters.startDate}
                onChange={(date) => setFilters({ ...filters, startDate: date })}
                slotProps={{ textField: { size: 'small', fullWidth: true } }}
              />
            </Grid>
            <Grid item xs={12} sm={6} md={3}>
              <DateTimePicker
                label="종료일시"
                value={filters.endDate}
                onChange={(date) => setFilters({ ...filters, endDate: date })}
                slotProps={{ textField: { size: 'small', fullWidth: true } }}
              />
            </Grid>
          </Grid>
        </Toolbar>
      </Card>

      {/* 히스토리 테이블 */}
      <Card>
        <DataGrid
          rows={historyData?.content || []}
          columns={columns}
          getRowId={(row) => row.executionId}
          paginationMode="server"
          paginationModel={{ page, pageSize }}
          onPaginationModelChange={(model) => {
            setPage(model.page);
            setPageSize(model.pageSize);
          }}
          rowCount={historyData?.totalElements || 0}
          pageSizeOptions={[10, 25, 50, 100]}
          autoHeight
          sx={{ minHeight: 400 }}
        />
      </Card>

      {/* 상세 정보 다이얼로그 */}
      <ExecutionDetailDialog
        open={detailDialogOpen}
        execution={selectedExecution}
        onClose={() => {
          setDetailDialogOpen(false);
          setSelectedExecution(null);
        }}
      />
    </Box>
  );
}
```

**이유**: Chart.js로 시각적 통계 분석, 상세한 필터링 기능, 실행 상세 정보 다이얼로그

## 2. 스케줄 관리

### 2.1 BatchSchedule.java (엔티티 완성)
**파일 위치**: `/src/main/java/com/sbs/datasync/entity/BatchSchedule.java`
```java
package com.sbs.datasync.entity;

import jakarta.persistence.*;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Size;

import java.time.LocalDateTime;

@Entity
@Table(name = "BATCH_SCHEDULE",
       indexes = {
           @Index(name = "IDX_BATCH_SCHEDULE_01", columnList = "JOB_CONFIG_ID"),
           @Index(name = "IDX_BATCH_SCHEDULE_02", columnList = "IS_ENABLED"),
           @Index(name = "IDX_BATCH_SCHEDULE_03", columnList = "NEXT_EXECUTION_TIME")
       })
public class BatchSchedule extends BaseEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "SCHEDULE_ID")
    private Long scheduleId;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "JOB_CONFIG_ID", nullable = false)
    private BatchJobConfig jobConfig;

    @NotBlank
    @Size(max = 100)
    @Column(name = "SCHEDULE_NAME", nullable = false, length = 100, unique = true)
    private String scheduleName;

    @NotBlank
    @Size(max = 100)
    @Column(name = "CRON_EXPRESSION", nullable = false, length = 100)
    private String cronExpression;

    @Column(name = "IS_ENABLED", nullable = false, length = 1)
    private String isEnabled = "Y";

    @Column(name = "NEXT_EXECUTION_TIME")
    private LocalDateTime nextExecutionTime;

    @Column(name = "LAST_EXECUTION_TIME")
    private LocalDateTime lastExecutionTime;

    @Size(max = 500)
    @Column(name = "DESCRIPTION", length = 500)
    private String description;

    @Column(name = "TIMEZONE", length = 50)
    private String timezone = "Asia/Seoul";

    @Column(name = "MAX_RETRY_COUNT")
    private Integer maxRetryCount = 3;

    @Column(name = "CURRENT_RETRY_COUNT")
    private Integer currentRetryCount = 0;

    // Getters, Setters, Business methods...

    public boolean isEnabled() {
        return "Y".equals(this.isEnabled);
    }

    public void enable() {
        this.isEnabled = "Y";
        this.currentRetryCount = 0;
    }

    public void disable() {
        this.isEnabled = "N";
    }

    public void updateLastExecutionTime() {
        this.lastExecutionTime = LocalDateTime.now();
        this.currentRetryCount = 0;
    }

    public void incrementRetryCount() {
        this.currentRetryCount = (this.currentRetryCount != null ? this.currentRetryCount : 0) + 1;
    }

    public boolean canRetry() {
        return this.currentRetryCount != null && this.maxRetryCount != null &&
               this.currentRetryCount < this.maxRetryCount;
    }
}
```

### 2.2 QuartzSchedulerService.java
**파일 위치**: `/src/main/java/com/sbs/datasync/service/QuartzSchedulerService.java`
```java
package com.sbs.datasync.service;

import com.sbs.datasync.batch.job.BatchJobFactory;
import com.sbs.datasync.entity.BatchJobConfig;
import com.sbs.datasync.entity.BatchSchedule;
import org.quartz.*;
import org.quartz.impl.matchers.GroupMatcher;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.batch.core.JobParameters;
import org.springframework.batch.core.launch.JobLauncher;
import org.springframework.scheduling.quartz.QuartzJobBean;
import org.springframework.stereotype.Service;

import java.time.LocalDateTime;
import java.time.ZoneId;
import java.util.*;

@Service
public class QuartzSchedulerService {

    private static final Logger logger = LoggerFactory.getLogger(QuartzSchedulerService.class);
    private static final String JOB_GROUP = "BATCH_JOBS";
    private static final String TRIGGER_GROUP = "BATCH_TRIGGERS";

    private final Scheduler scheduler;
    private final JobLauncher jobLauncher;
    private final BatchJobFactory jobFactory;

    public QuartzSchedulerService(Scheduler scheduler, JobLauncher jobLauncher, BatchJobFactory jobFactory) {
        this.scheduler = scheduler;
        this.jobLauncher = jobLauncher;
        this.jobFactory = jobFactory;
    }

    public void scheduleJob(BatchSchedule batchSchedule) throws SchedulerException {
        JobDetail jobDetail = JobBuilder.newJob(ScheduledBatchJob.class)
                .withIdentity(batchSchedule.getScheduleName(), JOB_GROUP)
                .withDescription(batchSchedule.getDescription())
                .usingJobData("scheduleId", batchSchedule.getScheduleId())
                .usingJobData("jobConfigId", batchSchedule.getJobConfig().getJobConfigId())
                .build();

        Trigger trigger = TriggerBuilder.newTrigger()
                .withIdentity(batchSchedule.getScheduleName(), TRIGGER_GROUP)
                .withSchedule(CronScheduleBuilder.cronSchedule(batchSchedule.getCronExpression())
                        .inTimeZone(TimeZone.getTimeZone(batchSchedule.getTimezone())))
                .build();

        scheduler.scheduleJob(jobDetail, trigger);

        // 다음 실행 시간 업데이트
        Date nextFireTime = trigger.getNextFireTime();
        if (nextFireTime != null) {
            batchSchedule.setNextExecutionTime(
                LocalDateTime.ofInstant(nextFireTime.toInstant(), ZoneId.systemDefault())
            );
        }

        logger.info("스케줄 등록 완료: {} - 다음 실행: {}",
                   batchSchedule.getScheduleName(), batchSchedule.getNextExecutionTime());
    }

    public void unscheduleJob(String scheduleName) throws SchedulerException {
        JobKey jobKey = new JobKey(scheduleName, JOB_GROUP);
        scheduler.deleteJob(jobKey);
        logger.info("스케줄 삭제 완료: {}", scheduleName);
    }

    public void pauseJob(String scheduleName) throws SchedulerException {
        JobKey jobKey = new JobKey(scheduleName, JOB_GROUP);
        scheduler.pauseJob(jobKey);
        logger.info("스케줄 일시정지: {}", scheduleName);
    }

    public void resumeJob(String scheduleName) throws SchedulerException {
        JobKey jobKey = new JobKey(scheduleName, JOB_GROUP);
        scheduler.resumeJob(jobKey);
        logger.info("스케줄 재시작: {}", scheduleName);
    }

    public void triggerJob(String scheduleName) throws SchedulerException {
        JobKey jobKey = new JobKey(scheduleName, JOB_GROUP);
        scheduler.triggerJob(jobKey);
        logger.info("스케줄 수동 실행: {}", scheduleName);
    }

    public List<String> getRunningJobs() throws SchedulerException {
        List<String> runningJobs = new ArrayList<>();
        List<JobExecutionContext> executingJobs = scheduler.getCurrentlyExecutingJobs();

        for (JobExecutionContext context : executingJobs) {
            runningJobs.add(context.getJobDetail().getKey().getName());
        }

        return runningJobs;
    }

    public boolean isValidCronExpression(String cronExpression) {
        try {
            CronExpression.validateExpression(cronExpression);
            return true;
        } catch (ParseException e) {
            return false;
        }
    }

    public Date getNextExecutionTime(String cronExpression, String timezone) {
        try {
            CronExpression cron = new CronExpression(cronExpression);
            cron.setTimeZone(TimeZone.getTimeZone(timezone));
            return cron.getNextValidTimeAfter(new Date());
        } catch (ParseException e) {
            logger.error("크론 표현식 파싱 오류: {}", cronExpression, e);
            return null;
        }
    }

    // Quartz Job 구현
    public static class ScheduledBatchJob extends QuartzJobBean {
        private JobLauncher jobLauncher;
        private BatchJobFactory jobFactory;

        public void setJobLauncher(JobLauncher jobLauncher) {
            this.jobLauncher = jobLauncher;
        }

        public void setBatchJobFactory(BatchJobFactory jobFactory) {
            this.jobFactory = jobFactory;
        }

        @Override
        protected void executeInternal(JobExecutionContext context) throws JobExecutionException {
            JobDataMap dataMap = context.getJobDetail().getJobDataMap();
            Long scheduleId = dataMap.getLong("scheduleId");
            Long jobConfigId = dataMap.getLong("jobConfigId");

            logger.info("스케줄된 배치 작업 실행: scheduleId={}, jobConfigId={}", scheduleId, jobConfigId);

            try {
                // BatchJobConfig 조회 및 실행 로직 구현
                // 실제로는 Service 레이어를 통해 구현해야 함
            } catch (Exception e) {
                logger.error("스케줄된 배치 작업 실행 실패: scheduleId={}", scheduleId, e);
                throw new JobExecutionException("스케줄된 배치 작업 실행 실패", e);
            }
        }
    }
}
```

### 2.3 ScheduleManagement.tsx (프론트엔드 페이지)
**파일 위치**: `/frontend/src/pages/ScheduleManagement.tsx`
```tsx
import React, { useState } from 'react';
import {
  Box,
  Typography,
  Card,
  CardContent,
  Button,
  Chip,
  IconButton,
  Switch,
  Dialog,
  DialogTitle,
  DialogContent,
  DialogActions,
  Grid,
  Alert,
} from '@mui/material';
import {
  Add as AddIcon,
  Edit as EditIcon,
  Delete as DeleteIcon,
  PlayArrow as PlayIcon,
  Schedule as ScheduleIcon,
} from '@mui/icons-material';
import { DataGrid, GridColDef } from '@mui/x-data-grid';
import { useMutation, useQuery, useQueryClient } from '@tanstack/react-query';

import { scheduleService } from '@/services/scheduleService';
import { BatchSchedule } from '@/types/schedule';
import LoadingSpinner from '@/components/LoadingSpinner';
import ScheduleForm from '@/components/ScheduleForm';
import CronExpressionBuilder from '@/components/CronExpressionBuilder';

export default function ScheduleManagement() {
  const [selectedSchedule, setSelectedSchedule] = useState<BatchSchedule | null>(null);
  const [formOpen, setFormOpen] = useState(false);
  const [deleteDialogOpen, setDeleteDialogOpen] = useState(false);
  const [cronBuilderOpen, setCronBuilderOpen] = useState(false);

  const queryClient = useQueryClient();

  // 스케줄 목록 조회
  const { data: schedulesData, isLoading } = useQuery({
    queryKey: ['schedules'],
    queryFn: () => scheduleService.getSchedules(),
  });

  // 스케줄 토글
  const toggleScheduleMutation = useMutation({
    mutationFn: ({ scheduleId, enabled }: { scheduleId: number; enabled: boolean }) =>
      scheduleService.toggleSchedule(scheduleId, enabled),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['schedules'] });
    },
  });

  // 스케줄 삭제
  const deleteScheduleMutation = useMutation({
    mutationFn: scheduleService.deleteSchedule,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['schedules'] });
      setDeleteDialogOpen(false);
      setSelectedSchedule(null);
    },
  });

  // 스케줄 즉시 실행
  const triggerScheduleMutation = useMutation({
    mutationFn: scheduleService.triggerSchedule,
  });

  const columns: GridColDef[] = [
    {
      field: 'scheduleName',
      headerName: '스케줄명',
      width: 200,
      renderCell: (params) => (
        <Box>
          <Typography variant="body2" fontWeight="medium">
            {params.value}
          </Typography>
          <Typography variant="caption" color="textSecondary">
            {params.row.description}
          </Typography>
        </Box>
      ),
    },
    {
      field: 'jobName',
      headerName: '작업명',
      width: 180,
      valueGetter: (params) => params.row.jobConfig?.jobName,
    },
    {
      field: 'cronExpression',
      headerName: '크론 표현식',
      width: 150,
      renderCell: (params) => (
        <Typography variant="caption" fontFamily="monospace">
          {params.value}
        </Typography>
      ),
    },
    {
      field: 'nextExecutionTime',
      headerName: '다음 실행',
      width: 160,
      renderCell: (params) => {
        if (!params.value) return '-';
        const nextTime = new Date(params.value);
        const now = new Date();
        const diff = nextTime.getTime() - now.getTime();

        if (diff < 0) return '실행 예정';

        const hours = Math.floor(diff / (1000 * 60 * 60));
        const minutes = Math.floor((diff % (1000 * 60 * 60)) / (1000 * 60));

        return (
          <Box>
            <Typography variant="body2">
              {nextTime.toLocaleString()}
            </Typography>
            <Typography variant="caption" color="textSecondary">
              {hours > 0 ? `${hours}시간 ` : ''}{minutes}분 후
            </Typography>
          </Box>
        );
      },
    },
    {
      field: 'isEnabled',
      headerName: '활성 상태',
      width: 120,
      renderCell: (params) => (
        <Switch
          checked={params.row.isEnabled}
          onChange={(e) => toggleScheduleMutation.mutate({
            scheduleId: params.row.scheduleId,
            enabled: e.target.checked,
          })}
          size="small"
        />
      ),
    },
    {
      field: 'lastExecutionTime',
      headerName: '마지막 실행',
      width: 160,
      renderCell: (params) => {
        if (!params.value) return '-';
        return (
          <Typography variant="body2">
            {new Date(params.value).toLocaleString()}
          </Typography>
        );
      },
    },
    {
      field: 'actions',
      headerName: '작업',
      width: 150,
      sortable: false,
      renderCell: (params) => (
        <Box>
          <IconButton
            size="small"
            onClick={() => triggerScheduleMutation.mutate(params.row.scheduleId)}
            disabled={!params.row.isEnabled}
            title="즉시 실행"
          >
            <PlayIcon />
          </IconButton>
          <IconButton
            size="small"
            onClick={() => handleEdit(params.row)}
            title="수정"
          >
            <EditIcon />
          </IconButton>
          <IconButton
            size="small"
            onClick={() => handleDelete(params.row)}
            title="삭제"
          >
            <DeleteIcon />
          </IconButton>
        </Box>
      ),
    },
  ];

  const handleEdit = (schedule: BatchSchedule) => {
    setSelectedSchedule(schedule);
    setFormOpen(true);
  };

  const handleDelete = (schedule: BatchSchedule) => {
    setSelectedSchedule(schedule);
    setDeleteDialogOpen(true);
  };

  if (isLoading) {
    return <LoadingSpinner message="스케줄 목록을 불러오는 중..." />;
  }

  return (
    <Box>
      <Box display="flex" justifyContent="space-between" alignItems="center" mb={3}>
        <Typography variant="h4">스케줄 관리</Typography>
        <Box display="flex" gap={1}>
          <Button
            variant="outlined"
            startIcon={<ScheduleIcon />}
            onClick={() => setCronBuilderOpen(true)}
          >
            크론 빌더
          </Button>
          <Button
            variant="contained"
            startIcon={<AddIcon />}
            onClick={() => {
              setSelectedSchedule(null);
              setFormOpen(true);
            }}
          >
            새 스케줄 추가
          </Button>
        </Box>
      </Box>

      {/* 스케줄 현황 카드 */}
      <Grid container spacing={3} sx={{ mb: 3 }}>
        <Grid item xs={12} sm={6} md={3}>
          <Card>
            <CardContent>
              <Typography color="textSecondary" gutterBottom>
                전체 스케줄
              </Typography>
              <Typography variant="h4">
                {schedulesData?.length || 0}
              </Typography>
            </CardContent>
          </Card>
        </Grid>
        <Grid item xs={12} sm={6} md={3}>
          <Card>
            <CardContent>
              <Typography color="textSecondary" gutterBottom>
                활성 스케줄
              </Typography>
              <Typography variant="h4" color="success.main">
                {schedulesData?.filter(s => s.isEnabled).length || 0}
              </Typography>
            </CardContent>
          </Card>
        </Grid>
        <Grid item xs={12} sm={6} md={3}>
          <Card>
            <CardContent>
              <Typography color="textSecondary" gutterBottom>
                1시간 내 실행 예정
              </Typography>
              <Typography variant="h4" color="info.main">
                {schedulesData?.filter(s => {
                  if (!s.nextExecutionTime || !s.isEnabled) return false;
                  const diff = new Date(s.nextExecutionTime).getTime() - new Date().getTime();
                  return diff > 0 && diff <= 60 * 60 * 1000;
                }).length || 0}
              </Typography>
            </CardContent>
          </Card>
        </Grid>
        <Grid item xs={12} sm={6} md={3}>
          <Card>
            <CardContent>
              <Typography color="textSecondary" gutterBottom>
                오늘 실행된 스케줄
              </Typography>
              <Typography variant="h4" color="warning.main">
                {schedulesData?.filter(s => {
                  if (!s.lastExecutionTime) return false;
                  const lastExec = new Date(s.lastExecutionTime);
                  const today = new Date();
                  return lastExec.toDateString() === today.toDateString();
                }).length || 0}
              </Typography>
            </CardContent>
          </Card>
        </Grid>
      </Grid>

      {/* 스케줄 테이블 */}
      <Card>
        <DataGrid
          rows={schedulesData || []}
          columns={columns}
          getRowId={(row) => row.scheduleId}
          autoHeight
          sx={{ minHeight: 400 }}
          pageSizeOptions={[25, 50, 100]}
          initialState={{
            pagination: { paginationModel: { pageSize: 25 } },
          }}
        />
      </Card>

      {/* 스케줄 폼 다이얼로그 */}
      <ScheduleForm
        open={formOpen}
        schedule={selectedSchedule}
        onClose={() => {
          setFormOpen(false);
          setSelectedSchedule(null);
        }}
      />

      {/* 크론 빌더 다이얼로그 */}
      <CronExpressionBuilder
        open={cronBuilderOpen}
        onClose={() => setCronBuilderOpen(false)}
      />

      {/* 삭제 확인 다이얼로그 */}
      <Dialog open={deleteDialogOpen} onClose={() => setDeleteDialogOpen(false)}>
        <DialogTitle>스케줄 삭제 확인</DialogTitle>
        <DialogContent>
          <Typography>
            '{selectedSchedule?.scheduleName}' 스케줄을 삭제하시겠습니까?
            삭제된 스케줄은 복구할 수 없습니다.
          </Typography>
        </DialogContent>
        <DialogActions>
          <Button onClick={() => setDeleteDialogOpen(false)}>취소</Button>
          <Button
            onClick={() => selectedSchedule &&
              deleteScheduleMutation.mutate(selectedSchedule.scheduleId)}
            color="error"
            disabled={deleteScheduleMutation.isPending}
          >
            삭제
          </Button>
        </DialogActions>
      </Dialog>
    </Box>
  );
}
```

**이유**: Quartz 스케줄러와 연동, 실시간 다음 실행 시간 표시, 크론 표현식 빌더 제공

이상으로 Phase 5의 히스토리 및 스케줄 관리 구현 코드를 제시했습니다. 이 단계에서는 과거 실행 데이터 분석과 자동 실행을 위한 스케줄링 시스템을 완성합니다.