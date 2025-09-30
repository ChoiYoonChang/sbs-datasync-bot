# Phase 4: 핵심 UI 기능 - 구현 코드

## 개요
배치 작업 관리, 실행, 결과 표시를 위한 핵심 UI 컴포넌트들을 구현합니다.

## 1. 배치 작업 관리 화면

### 1.1 BatchJobList.tsx (완전 구현)
**파일 위치**: `/frontend/src/pages/BatchJobList.tsx`
```tsx
import React, { useState, useCallback } from 'react';
import {
  Box,
  Typography,
  Button,
  Card,
  Chip,
  IconButton,
  Menu,
  MenuItem,
  Dialog,
  DialogTitle,
  DialogContent,
  DialogActions,
  Alert,
  Toolbar,
  TextField,
  FormControl,
  InputLabel,
  Select,
} from '@mui/material';
import {
  Add as AddIcon,
  PlayArrow as PlayIcon,
  Edit as EditIcon,
  Delete as DeleteIcon,
  MoreVert as MoreIcon,
  Stop as StopIcon,
  Refresh as RefreshIcon,
} from '@mui/icons-material';
import { DataGrid, GridColDef, GridRowSelectionModel } from '@mui/x-data-grid';
import { useMutation, useQuery, useQueryClient } from '@tanstack/react-query';
import { format } from 'date-fns';
import { ko } from 'date-fns/locale';

import { batchService } from '@/services/batchService';
import { BatchJobConfig } from '@/types/batch';
import { useBatchStore } from '@/stores/batchStore';
import LoadingSpinner from '@/components/LoadingSpinner';
import BatchJobForm from '@/components/BatchJobForm';
import ExecutionProgress from '@/components/ExecutionProgress';

export default function BatchJobList() {
  const [page, setPage] = useState(0);
  const [pageSize, setPageSize] = useState(25);
  const [filters, setFilters] = useState({
    jobName: '',
    schemaName: 'FAS',
    tableName: '',
    isActive: true,
  });

  const [selectedJob, setSelectedJob] = useState<BatchJobConfig | null>(null);
  const [formOpen, setFormOpen] = useState(false);
  const [deleteDialogOpen, setDeleteDialogOpen] = useState(false);
  const [menuAnchor, setMenuAnchor] = useState<null | HTMLElement>(null);

  const queryClient = useQueryClient();
  const { selectedJobs, setSelectedJobs, executionStatuses } = useBatchStore();

  // 데이터 조회
  const { data: jobsData, isLoading, error } = useQuery({
    queryKey: ['batch-jobs', page, pageSize, filters],
    queryFn: () => batchService.getJobs({
      page,
      size: pageSize,
      ...filters,
    }),
  });

  // 작업 생성/수정
  const createJobMutation = useMutation({
    mutationFn: batchService.createJob,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['batch-jobs'] });
      setFormOpen(false);
      setSelectedJob(null);
    },
  });

  const updateJobMutation = useMutation({
    mutationFn: ({ id, data }: { id: number; data: any }) =>
      batchService.updateJob(id, data),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['batch-jobs'] });
      setFormOpen(false);
      setSelectedJob(null);
    },
  });

  // 작업 삭제
  const deleteJobMutation = useMutation({
    mutationFn: batchService.deleteJob,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['batch-jobs'] });
      setDeleteDialogOpen(false);
      setSelectedJob(null);
    },
  });

  // 작업 실행
  const executeJobMutation = useMutation({
    mutationFn: ({ jobId, executedBy }: { jobId: number; executedBy?: string }) =>
      batchService.executeJobAsync(jobId, executedBy),
    onSuccess: (data, variables) => {
      // 실행 상태 업데이트
      useBatchStore.getState().updateExecutionStatus(variables.jobId, {
        status: 'STARTED',
        executionId: data.executionId,
        startTime: new Date().toISOString(),
      });
    },
  });

  // 컬럼 정의
  const columns: GridColDef[] = [
    {
      field: 'jobName',
      headerName: '작업명',
      width: 200,
      renderCell: (params) => (
        <Box>
          <Typography variant="body2" fontWeight="medium">
            {params.value}
          </Typography>
          {params.row.memo && (
            <Typography variant="caption" color="textSecondary">
              {params.row.memo}
            </Typography>
          )}
        </Box>
      ),
    },
    {
      field: 'tableName',
      headerName: '테이블',
      width: 150,
      renderCell: (params) => (
        <Typography variant="body2">
          {params.row.schemaName}.{params.value}
        </Typography>
      ),
    },
    {
      field: 'dateRange',
      headerName: '기간',
      width: 200,
      renderCell: (params) => {
        const { startDate, endDate } = params.row;
        if (!startDate && !endDate) return '-';

        const start = startDate ? format(new Date(startDate), 'yy.MM.dd', { locale: ko }) : '';
        const end = endDate ? format(new Date(endDate), 'yy.MM.dd', { locale: ko }) : '';

        if (start && end) return `${start} ~ ${end}`;
        if (start) return `${start} ~`;
        if (end) return `~ ${end}`;
        return '-';
      },
    },
    {
      field: 'isActive',
      headerName: '상태',
      width: 100,
      renderCell: (params) => (
        <Chip
          label={params.value ? '활성' : '비활성'}
          color={params.value ? 'success' : 'default'}
          size="small"
        />
      ),
    },
    {
      field: 'execution',
      headerName: '실행 상태',
      width: 200,
      renderCell: (params) => {
        const status = executionStatuses.get(params.row.jobConfigId);
        if (!status) return '-';

        return <ExecutionProgress execution={status} compact />;
      },
    },
    {
      field: 'createdDate',
      headerName: '생성일',
      width: 120,
      renderCell: (params) => (
        <Typography variant="caption">
          {format(new Date(params.value), 'yy.MM.dd HH:mm', { locale: ko })}
        </Typography>
      ),
    },
    {
      field: 'actions',
      headerName: '작업',
      width: 120,
      sortable: false,
      renderCell: (params) => (
        <Box>
          <IconButton
            size="small"
            onClick={() => handleExecuteJob(params.row)}
            disabled={!params.row.isActive || executeJobMutation.isPending}
          >
            <PlayIcon />
          </IconButton>
          <IconButton
            size="small"
            onClick={(e) => handleMenuOpen(e, params.row)}
          >
            <MoreIcon />
          </IconButton>
        </Box>
      ),
    },
  ];

  const handleExecuteJob = useCallback((job: BatchJobConfig) => {
    executeJobMutation.mutate({
      jobId: job.jobConfigId,
      executedBy: 'user', // TODO: 실제 사용자 정보 가져오기
    });
  }, [executeJobMutation]);

  const handleMenuOpen = (event: React.MouseEvent<HTMLElement>, job: BatchJobConfig) => {
    setMenuAnchor(event.currentTarget);
    setSelectedJob(job);
  };

  const handleMenuClose = () => {
    setMenuAnchor(null);
    setSelectedJob(null);
  };

  const handleEdit = () => {
    setFormOpen(true);
    handleMenuClose();
  };

  const handleDelete = () => {
    setDeleteDialogOpen(true);
    handleMenuClose();
  };

  const handleBulkExecute = () => {
    selectedJobs.forEach(jobId => {
      const job = jobsData?.content.find(j => j.jobConfigId === jobId);
      if (job && job.isActive) {
        executeJobMutation.mutate({ jobId, executedBy: 'user' });
      }
    });
  };

  if (isLoading) {
    return <LoadingSpinner message="배치 작업 목록을 불러오는 중..." />;
  }

  if (error) {
    return (
      <Alert severity="error">
        배치 작업 목록을 불러오는데 실패했습니다.
      </Alert>
    );
  }

  return (
    <Box>
      {/* 헤더 */}
      <Box display="flex" justifyContent="space-between" alignItems="center" mb={3}>
        <Typography variant="h4">배치 작업 관리</Typography>
        <Box display="flex" gap={1}>
          <Button
            variant="outlined"
            startIcon={<RefreshIcon />}
            onClick={() => queryClient.invalidateQueries({ queryKey: ['batch-jobs'] })}
          >
            새로고침
          </Button>
          <Button
            variant="contained"
            startIcon={<AddIcon />}
            onClick={() => {
              setSelectedJob(null);
              setFormOpen(true);
            }}
          >
            새 작업 추가
          </Button>
        </Box>
      </Box>

      {/* 필터 */}
      <Card sx={{ mb: 2 }}>
        <Toolbar>
          <Box display="flex" gap={2} flexWrap="wrap" width="100%">
            <TextField
              label="작업명"
              size="small"
              value={filters.jobName}
              onChange={(e) => setFilters({ ...filters, jobName: e.target.value })}
            />
            <FormControl size="small" sx={{ minWidth: 120 }}>
              <InputLabel>스키마</InputLabel>
              <Select
                value={filters.schemaName}
                label="스키마"
                onChange={(e) => setFilters({ ...filters, schemaName: e.target.value })}
              >
                <MenuItem value="FAS">FAS</MenuItem>
                <MenuItem value="HR">HR</MenuItem>
                <MenuItem value="MFG">MFG</MenuItem>
              </Select>
            </FormControl>
            <TextField
              label="테이블명"
              size="small"
              value={filters.tableName}
              onChange={(e) => setFilters({ ...filters, tableName: e.target.value })}
            />
          </Box>
        </Toolbar>
      </Card>

      {/* 선택된 작업에 대한 액션 */}
      {selectedJobs.length > 0 && (
        <Card sx={{ mb: 2 }}>
          <Toolbar>
            <Typography variant="body2" sx={{ flexGrow: 1 }}>
              {selectedJobs.length}개 작업이 선택됨
            </Typography>
            <Button
              variant="contained"
              startIcon={<PlayIcon />}
              onClick={handleBulkExecute}
              disabled={executeJobMutation.isPending}
            >
              선택 작업 실행
            </Button>
          </Toolbar>
        </Card>
      )}

      {/* 데이터 그리드 */}
      <Card>
        <DataGrid
          rows={jobsData?.content || []}
          columns={columns}
          getRowId={(row) => row.jobConfigId}
          checkboxSelection
          rowSelectionModel={selectedJobs}
          onRowSelectionModelChange={(selection: GridRowSelectionModel) => {
            setSelectedJobs(selection as number[]);
          }}
          paginationMode="server"
          paginationModel={{ page, pageSize }}
          onPaginationModelChange={(model) => {
            setPage(model.page);
            setPageSize(model.pageSize);
          }}
          rowCount={jobsData?.totalElements || 0}
          pageSizeOptions={[10, 25, 50]}
          disableRowSelectionOnClick
          autoHeight
          sx={{ minHeight: 400 }}
        />
      </Card>

      {/* 컨텍스트 메뉴 */}
      <Menu
        anchorEl={menuAnchor}
        open={Boolean(menuAnchor)}
        onClose={handleMenuClose}
      >
        <MenuItem onClick={handleEdit}>
          <EditIcon sx={{ mr: 1 }} fontSize="small" />
          수정
        </MenuItem>
        <MenuItem onClick={handleDelete}>
          <DeleteIcon sx={{ mr: 1 }} fontSize="small" />
          삭제
        </MenuItem>
      </Menu>

      {/* 작업 폼 다이얼로그 */}
      <BatchJobForm
        open={formOpen}
        job={selectedJob}
        onClose={() => {
          setFormOpen(false);
          setSelectedJob(null);
        }}
        onSubmit={(data) => {
          if (selectedJob) {
            updateJobMutation.mutate({ id: selectedJob.jobConfigId, data });
          } else {
            createJobMutation.mutate(data);
          }
        }}
        loading={createJobMutation.isPending || updateJobMutation.isPending}
      />

      {/* 삭제 확인 다이얼로그 */}
      <Dialog open={deleteDialogOpen} onClose={() => setDeleteDialogOpen(false)}>
        <DialogTitle>작업 삭제 확인</DialogTitle>
        <DialogContent>
          <Typography>
            '{selectedJob?.jobName}' 작업을 삭제하시겠습니까?
            삭제된 작업은 복구할 수 없습니다.
          </Typography>
        </DialogContent>
        <DialogActions>
          <Button onClick={() => setDeleteDialogOpen(false)}>취소</Button>
          <Button
            onClick={() => selectedJob && deleteJobMutation.mutate(selectedJob.jobConfigId)}
            color="error"
            disabled={deleteJobMutation.isPending}
          >
            삭제
          </Button>
        </DialogActions>
      </Dialog>
    </Box>
  );
}
```

**이유**: DataGrid로 대용량 데이터 처리, 다중 선택과 일괄 작업, 실시간 상태 업데이트

## 2. 배치 작업 폼

### 2.1 BatchJobForm.tsx
**파일 위치**: `/frontend/src/components/BatchJobForm.tsx`
```tsx
import React, { useEffect, useState } from 'react';
import {
  Dialog,
  DialogTitle,
  DialogContent,
  DialogActions,
  TextField,
  Button,
  Grid,
  FormControl,
  InputLabel,
  Select,
  MenuItem,
  Autocomplete,
  Box,
  Typography,
  Alert,
} from '@mui/material';
import { DatePicker } from '@mui/x-date-pickers/DatePicker';
import { useQuery } from '@tanstack/react-query';
import { useForm, Controller } from 'react-hook-form';
import { yupResolver } from '@hookform/resolvers/yup';
import * as yup from 'yup';

import { BatchJobConfig } from '@/types/batch';
import { batchService } from '@/services/batchService';
import CodeEditor from '@/components/CodeEditor';

const schema = yup.object({
  jobName: yup.string().required('작업명은 필수입니다').max(100, '작업명은 100자를 초과할 수 없습니다'),
  schemaName: yup.string().required('스키마명은 필수입니다'),
  tableName: yup.string().required('테이블명은 필수입니다').max(100, '테이블명은 100자를 초과할 수 없습니다'),
  startDate: yup.date().nullable(),
  endDate: yup.date().nullable(),
  insertQuery: yup.string().max(4000, '쿼리는 4000자를 초과할 수 없습니다'),
  memo: yup.string().max(500, '메모는 500자를 초과할 수 없습니다'),
});

interface BatchJobFormProps {
  open: boolean;
  job?: BatchJobConfig | null;
  onClose: () => void;
  onSubmit: (data: any) => void;
  loading?: boolean;
}

export default function BatchJobForm({
  open,
  job,
  onClose,
  onSubmit,
  loading = false
}: BatchJobFormProps) {
  const [selectedSchema, setSelectedSchema] = useState('FAS');

  const { control, handleSubmit, reset, watch, formState: { errors } } = useForm({
    resolver: yupResolver(schema),
    defaultValues: {
      jobName: '',
      schemaName: 'FAS',
      tableName: '',
      startDate: null,
      endDate: null,
      insertQuery: '',
      memo: '',
    },
  });

  const schemaNameWatch = watch('schemaName');

  // 테이블 목록 조회
  const { data: tableNames = [] } = useQuery({
    queryKey: ['table-names', schemaNameWatch],
    queryFn: () => batchService.getTableNames(schemaNameWatch),
    enabled: !!schemaNameWatch && open,
  });

  useEffect(() => {
    if (job) {
      reset({
        jobName: job.jobName,
        schemaName: job.schemaName,
        tableName: job.tableName,
        startDate: job.startDate ? new Date(job.startDate) : null,
        endDate: job.endDate ? new Date(job.endDate) : null,
        insertQuery: job.insertQuery || '',
        memo: job.memo || '',
      });
      setSelectedSchema(job.schemaName);
    } else {
      reset({
        jobName: '',
        schemaName: 'FAS',
        tableName: '',
        startDate: null,
        endDate: null,
        insertQuery: '',
        memo: '',
      });
      setSelectedSchema('FAS');
    }
  }, [job, reset, open]);

  const handleFormSubmit = (data: any) => {
    const submitData = {
      ...data,
      startDate: data.startDate ? data.startDate.toISOString().split('T')[0] : null,
      endDate: data.endDate ? data.endDate.toISOString().split('T')[0] : null,
    };
    onSubmit(submitData);
  };

  return (
    <Dialog open={open} onClose={onClose} maxWidth="md" fullWidth>
      <form onSubmit={handleSubmit(handleFormSubmit)}>
        <DialogTitle>
          {job ? '배치 작업 수정' : '새 배치 작업 추가'}
        </DialogTitle>

        <DialogContent>
          <Grid container spacing={3} sx={{ mt: 1 }}>
            {/* 작업명 */}
            <Grid item xs={12}>
              <Controller
                name="jobName"
                control={control}
                render={({ field }) => (
                  <TextField
                    {...field}
                    label="작업명"
                    fullWidth
                    required
                    error={!!errors.jobName}
                    helperText={errors.jobName?.message}
                  />
                )}
              />
            </Grid>

            {/* 스키마명 */}
            <Grid item xs={12} sm={6}>
              <Controller
                name="schemaName"
                control={control}
                render={({ field }) => (
                  <FormControl fullWidth required>
                    <InputLabel>스키마명</InputLabel>
                    <Select {...field} label="스키마명">
                      <MenuItem value="FAS">FAS (재무회계)</MenuItem>
                      <MenuItem value="HR">HR (인사)</MenuItem>
                      <MenuItem value="MFG">MFG (제작)</MenuItem>
                    </Select>
                  </FormControl>
                )}
              />
            </Grid>

            {/* 테이블명 */}
            <Grid item xs={12} sm={6}>
              <Controller
                name="tableName"
                control={control}
                render={({ field: { onChange, value, ...field } }) => (
                  <Autocomplete
                    {...field}
                    value={value || ''}
                    onChange={(_, newValue) => onChange(newValue || '')}
                    options={tableNames}
                    freeSolo
                    renderInput={(params) => (
                      <TextField
                        {...params}
                        label="테이블명"
                        required
                        error={!!errors.tableName}
                        helperText={errors.tableName?.message}
                      />
                    )}
                  />
                )}
              />
            </Grid>

            {/* 기간 설정 */}
            <Grid item xs={12}>
              <Typography variant="subtitle2" gutterBottom>
                데이터 범위 (선택사항)
              </Typography>
            </Grid>
            <Grid item xs={12} sm={6}>
              <Controller
                name="startDate"
                control={control}
                render={({ field }) => (
                  <DatePicker
                    {...field}
                    label="시작일"
                    slotProps={{
                      textField: {
                        fullWidth: true,
                        error: !!errors.startDate,
                        helperText: errors.startDate?.message,
                      },
                    }}
                  />
                )}
              />
            </Grid>
            <Grid item xs={12} sm={6}>
              <Controller
                name="endDate"
                control={control}
                render={({ field }) => (
                  <DatePicker
                    {...field}
                    label="종료일"
                    slotProps={{
                      textField: {
                        fullWidth: true,
                        error: !!errors.endDate,
                        helperText: errors.endDate?.message,
                      },
                    }}
                  />
                )}
              />
            </Grid>

            {/* 커스텀 쿼리 */}
            <Grid item xs={12}>
              <Typography variant="subtitle2" gutterBottom>
                사용자 정의 쿼리 (선택사항)
              </Typography>
              <Controller
                name="insertQuery"
                control={control}
                render={({ field }) => (
                  <CodeEditor
                    {...field}
                    language="sql"
                    placeholder="SELECT * FROM FAS.TABLE_NAME WHERE ..."
                    height="200px"
                    error={!!errors.insertQuery}
                    helperText={errors.insertQuery?.message}
                  />
                )}
              />
            </Grid>

            {/* 메모 */}
            <Grid item xs={12}>
              <Controller
                name="memo"
                control={control}
                render={({ field }) => (
                  <TextField
                    {...field}
                    label="메모"
                    fullWidth
                    multiline
                    rows={3}
                    error={!!errors.memo}
                    helperText={errors.memo?.message}
                  />
                )}
              />
            </Grid>
          </Grid>

          {Object.keys(errors).length > 0 && (
            <Alert severity="error" sx={{ mt: 2 }}>
              입력값을 확인해주세요.
            </Alert>
          )}
        </DialogContent>

        <DialogActions>
          <Button onClick={onClose}>취소</Button>
          <Button
            type="submit"
            variant="contained"
            disabled={loading}
          >
            {loading ? '저장 중...' : (job ? '수정' : '추가')}
          </Button>
        </DialogActions>
      </form>
    </Dialog>
  );
}
```

**이유**: React Hook Form으로 폼 상태 관리, Yup 스키마로 유효성 검증, Autocomplete로 UX 향상

## 3. 실행 진행률 컴포넌트

### 3.1 ExecutionProgress.tsx
**파일 위치**: `/frontend/src/components/ExecutionProgress.tsx`
```tsx
import React from 'react';
import {
  Box,
  LinearProgress,
  Typography,
  Chip,
  Tooltip,
  IconButton,
} from '@mui/material';
import {
  CheckCircle as CheckIcon,
  Error as ErrorIcon,
  PlayArrow as PlayIcon,
  Stop as StopIcon,
} from '@mui/icons-material';
import { ExecutionStatus } from '@/types/batch';

interface ExecutionProgressProps {
  execution: {
    status: ExecutionStatus;
    executionId?: number;
    startTime?: string;
    progress?: number;
    totalCount?: number;
    successCount?: number;
    errorCount?: number;
  };
  compact?: boolean;
  onStop?: (executionId: number) => void;
}

const statusConfig = {
  STARTED: { color: 'info', icon: PlayIcon, label: '실행 중' },
  COMPLETED: { color: 'success', icon: CheckIcon, label: '완료' },
  FAILED: { color: 'error', icon: ErrorIcon, label: '실패' },
  STOPPED: { color: 'default', icon: StopIcon, label: '중단' },
  ABANDONED: { color: 'warning', icon: ErrorIcon, label: '포기' },
} as const;

export default function ExecutionProgress({
  execution,
  compact = false,
  onStop
}: ExecutionProgressProps) {
  const config = statusConfig[execution.status];
  const Icon = config.icon;

  const progress = execution.progress || 0;
  const isRunning = execution.status === 'STARTED';

  if (compact) {
    return (
      <Box display="flex" alignItems="center" gap={1}>
        <Chip
          icon={<Icon />}
          label={config.label}
          color={config.color}
          size="small"
        />
        {isRunning && (
          <Typography variant="caption">
            {progress}%
          </Typography>
        )}
      </Box>
    );
  }

  return (
    <Box>
      <Box display="flex" alignItems="center" justifyContent="space-between" mb={1}>
        <Box display="flex" alignItems="center" gap={1}>
          <Icon color={config.color} />
          <Typography variant="body2" fontWeight="medium">
            {config.label}
          </Typography>
          {execution.executionId && (
            <Typography variant="caption" color="textSecondary">
              (ID: {execution.executionId})
            </Typography>
          )}
        </Box>

        {isRunning && onStop && execution.executionId && (
          <Tooltip title="실행 중단">
            <IconButton
              size="small"
              onClick={() => onStop(execution.executionId!)}
            >
              <StopIcon />
            </IconButton>
          </Tooltip>
        )}
      </Box>

      {isRunning && (
        <>
          <LinearProgress
            variant="determinate"
            value={progress}
            sx={{ mb: 1 }}
          />
          <Box display="flex" justifyContent="space-between">
            <Typography variant="caption">
              진행률: {progress}%
            </Typography>
            <Typography variant="caption">
              {execution.successCount || 0} / {execution.totalCount || 0} 건
            </Typography>
          </Box>
        </>
      )}

      {execution.status === 'COMPLETED' && (
        <Typography variant="caption" color="success.main">
          {execution.successCount}건 성공
          {execution.errorCount && execution.errorCount > 0 && (
            <>, {execution.errorCount}건 오류</>
          )}
        </Typography>
      )}

      {execution.status === 'FAILED' && (
        <Typography variant="caption" color="error.main">
          실행 실패
        </Typography>
      )}

      {execution.startTime && (
        <Typography variant="caption" color="textSecondary" display="block">
          시작: {new Date(execution.startTime).toLocaleString()}
        </Typography>
      )}
    </Box>
  );
}
```

**이유**: 실행 상태별 시각적 피드백, 진행률 표시, 실행 제어 기능 통합

## 4. 코드 에디터 컴포넌트

### 4.1 CodeEditor.tsx
**파일 위치**: `/frontend/src/components/CodeEditor.tsx`
```tsx
import React, { forwardRef } from 'react';
import { Box, TextField, Typography } from '@mui/material';

interface CodeEditorProps {
  value: string;
  onChange: (value: string) => void;
  language?: string;
  placeholder?: string;
  height?: string;
  error?: boolean;
  helperText?: string;
}

// 간단한 코드 에디터 (Monaco Editor 대신 기본 textarea 사용)
// 실제 구현에서는 @monaco-editor/react 사용 권장
const CodeEditor = forwardRef<HTMLTextAreaElement, CodeEditorProps>(({
  value,
  onChange,
  language = 'sql',
  placeholder,
  height = '150px',
  error = false,
  helperText,
}, ref) => {
  return (
    <Box>
      <TextField
        ref={ref}
        value={value}
        onChange={(e) => onChange(e.target.value)}
        placeholder={placeholder}
        multiline
        fullWidth
        error={error}
        helperText={helperText}
        InputProps={{
          sx: {
            fontFamily: 'Monaco, Consolas, "Liberation Mono", "Courier New", monospace',
            fontSize: '14px',
            minHeight: height,
          },
        }}
        sx={{
          '& .MuiInputBase-root': {
            alignItems: 'flex-start',
          },
        }}
      />
      {language === 'sql' && (
        <Typography variant="caption" color="textSecondary">
          SQL 쿼리를 입력하세요. 예: SELECT * FROM FAS.TABLE_NAME WHERE CREATE_DATE >= '2023-01-01'
        </Typography>
      )}
    </Box>
  );
});

CodeEditor.displayName = 'CodeEditor';

export default CodeEditor;
```

**이유**: SQL 쿼리 입력을 위한 전용 에디터, 향후 Monaco Editor로 확장 가능

이상으로 Phase 4의 핵심 UI 기능 구현 코드를 제시했습니다. 이 단계에서는 실제 사용자가 상호작용하는 주요 화면들을 완성합니다.