# Phase 3: 프론트엔드 기본 구조 - 구현 코드

## 개요
React + TypeScript + Material-UI 기반의 프론트엔드 기본 구조를 설정합니다.

## 1. 프로젝트 초기 설정

### 1.1 tsconfig.json
**파일 위치**: `/frontend/tsconfig.json`
```json
{
  "compilerOptions": {
    "target": "ES2020",
    "useDefineForClassFields": true,
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "skipLibCheck": true,
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "jsx": "react-jsx",
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"],
      "@/components/*": ["src/components/*"],
      "@/pages/*": ["src/pages/*"],
      "@/hooks/*": ["src/hooks/*"],
      "@/services/*": ["src/services/*"],
      "@/types/*": ["src/types/*"],
      "@/utils/*": ["src/utils/*"]
    }
  },
  "include": ["src"],
  "references": [{ "path": "./tsconfig.node.json" }]
}
```

**이유**: 절대 경로 import로 가독성 향상, 엄격한 TypeScript 설정으로 타입 안전성 확보

### 1.2 App.tsx
**파일 위치**: `/frontend/src/App.tsx`
```tsx
import { BrowserRouter as Router, Routes, Route } from 'react-router-dom';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { ThemeProvider, createTheme } from '@mui/material/styles';
import CssBaseline from '@mui/material/CssBaseline';
import { LocalizationProvider } from '@mui/x-date-pickers/LocalizationProvider';
import { AdapterDateFns } from '@mui/x-date-pickers/AdapterDateFns';
import { ko } from 'date-fns/locale';

import Layout from '@/components/Layout';
import Dashboard from '@/pages/Dashboard';
import BatchJobList from '@/pages/BatchJobList';
import ExecutionHistory from '@/pages/ExecutionHistory';
import ScheduleManagement from '@/pages/ScheduleManagement';
import Settings from '@/pages/Settings';
import { useThemeStore } from '@/stores/themeStore';

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      retry: 1,
      refetchOnWindowFocus: false,
      staleTime: 5 * 60 * 1000, // 5분
    },
  },
});

function App() {
  const { darkMode } = useThemeStore();

  const theme = createTheme({
    palette: {
      mode: darkMode ? 'dark' : 'light',
      primary: {
        main: '#1976d2',
      },
      secondary: {
        main: '#dc004e',
      },
    },
    typography: {
      fontFamily: [
        '-apple-system',
        'BlinkMacSystemFont',
        '"Segoe UI"',
        'Roboto',
        '"Helvetica Neue"',
        'Arial',
        'sans-serif',
        '"Apple Color Emoji"',
        '"Segoe UI Emoji"',
        '"Segoe UI Symbol"',
      ].join(','),
    },
  });

  return (
    <QueryClientProvider client={queryClient}>
      <ThemeProvider theme={theme}>
        <LocalizationProvider dateAdapter={AdapterDateFns} adapterLocale={ko}>
          <CssBaseline />
          <Router>
            <Layout>
              <Routes>
                <Route path="/" element={<Dashboard />} />
                <Route path="/jobs" element={<BatchJobList />} />
                <Route path="/history" element={<ExecutionHistory />} />
                <Route path="/schedule" element={<ScheduleManagement />} />
                <Route path="/settings" element={<Settings />} />
              </Routes>
            </Layout>
          </Router>
        </LocalizationProvider>
      </ThemeProvider>
    </QueryClientProvider>
  );
}

export default App;
```

**이유**: React Query로 서버 상태 관리, Material-UI 테마 시스템, 한국 로케일 적용

## 2. 상태 관리

### 2.1 themeStore.ts
**파일 위치**: `/frontend/src/stores/themeStore.ts`
```typescript
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

interface ThemeState {
  darkMode: boolean;
  toggleDarkMode: () => void;
  setDarkMode: (darkMode: boolean) => void;
}

export const useThemeStore = create<ThemeState>()(
  persist(
    (set) => ({
      darkMode: false,
      toggleDarkMode: () => set((state) => ({ darkMode: !state.darkMode })),
      setDarkMode: (darkMode: boolean) => set({ darkMode }),
    }),
    {
      name: 'theme-storage',
    }
  )
);
```

### 2.2 batchStore.ts
**파일 위치**: `/frontend/src/stores/batchStore.ts`
```typescript
import { create } from 'zustand';
import { BatchJobConfig, ExecutionStatus } from '@/types/batch';

interface BatchState {
  selectedJobs: number[];
  executionStatuses: Map<number, ExecutionStatus>;
  setSelectedJobs: (jobs: number[]) => void;
  toggleJobSelection: (jobId: number) => void;
  clearSelection: () => void;
  updateExecutionStatus: (jobId: number, status: ExecutionStatus) => void;
  clearExecutionStatus: (jobId: number) => void;
}

export const useBatchStore = create<BatchState>((set) => ({
  selectedJobs: [],
  executionStatuses: new Map(),
  setSelectedJobs: (jobs) => set({ selectedJobs: jobs }),
  toggleJobSelection: (jobId) =>
    set((state) => ({
      selectedJobs: state.selectedJobs.includes(jobId)
        ? state.selectedJobs.filter((id) => id !== jobId)
        : [...state.selectedJobs, jobId],
    })),
  clearSelection: () => set({ selectedJobs: [] }),
  updateExecutionStatus: (jobId, status) =>
    set((state) => ({
      executionStatuses: new Map(state.executionStatuses).set(jobId, status),
    })),
  clearExecutionStatus: (jobId) =>
    set((state) => {
      const newMap = new Map(state.executionStatuses);
      newMap.delete(jobId);
      return { executionStatuses: newMap };
    }),
}));
```

**이유**: Zustand로 간단한 상태 관리, 타입 안전성 보장, 로컬 스토리지 연동

## 3. API 서비스

### 3.1 api.ts
**파일 위치**: `/frontend/src/services/api.ts`
```typescript
import axios from 'axios';

const API_BASE_URL = import.meta.env.VITE_API_BASE_URL || '/api';

export const api = axios.create({
  baseURL: API_BASE_URL,
  timeout: 30000,
  headers: {
    'Content-Type': 'application/json',
  },
});

// Request interceptor
api.interceptors.request.use(
  (config) => {
    const token = localStorage.getItem('auth-token');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => {
    return Promise.reject(error);
  }
);

// Response interceptor
api.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      localStorage.removeItem('auth-token');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);

export default api;
```

### 3.2 batchService.ts
**파일 위치**: `/frontend/src/services/batchService.ts`
```typescript
import api from './api';
import { BatchJobConfig, BatchExecution, PaginatedResponse } from '@/types/batch';

export interface CreateBatchJobRequest {
  jobName: string;
  schemaName: string;
  tableName: string;
  startDate?: string;
  endDate?: string;
  insertQuery?: string;
  memo?: string;
}

export interface UpdateBatchJobRequest extends CreateBatchJobRequest {
  isActive: boolean;
}

class BatchService {
  // Job Config CRUD
  async getJobs(params?: {
    page?: number;
    size?: number;
    jobName?: string;
    schemaName?: string;
    tableName?: string;
    isActive?: boolean;
  }): Promise<PaginatedResponse<BatchJobConfig>> {
    const response = await api.get('/batch/jobs', { params });
    return response.data;
  }

  async getJob(jobId: number): Promise<BatchJobConfig> {
    const response = await api.get(`/batch/jobs/${jobId}`);
    return response.data;
  }

  async createJob(jobData: CreateBatchJobRequest): Promise<BatchJobConfig> {
    const response = await api.post('/batch/jobs', jobData);
    return response.data;
  }

  async updateJob(jobId: number, jobData: UpdateBatchJobRequest): Promise<BatchJobConfig> {
    const response = await api.put(`/batch/jobs/${jobId}`, jobData);
    return response.data;
  }

  async deleteJob(jobId: number): Promise<void> {
    await api.delete(`/batch/jobs/${jobId}`);
  }

  async toggleJobStatus(jobId: number, status: 'Y' | 'N'): Promise<BatchJobConfig> {
    const response = await api.patch(`/batch/jobs/${jobId}/status?status=${status}`);
    return response.data;
  }

  // Job Execution
  async executeJob(jobId: number, executedBy?: string): Promise<BatchExecution> {
    const params = executedBy ? { executedBy } : {};
    const response = await api.post(`/batch/execution/run/${jobId}`, null, { params });
    return response.data;
  }

  async executeJobAsync(jobId: number, executedBy?: string): Promise<{ executionId: number }> {
    const params = executedBy ? { executedBy } : {};
    const response = await api.post(`/batch/execution/run-async/${jobId}`, null, { params });
    return response.data;
  }

  async executeMultipleJobs(jobIds: number[], executedBy?: string): Promise<BatchExecution[]> {
    const response = await api.post('/batch/execution/run-multiple', {
      jobConfigIds: jobIds,
      executedBy,
    });
    return response.data;
  }

  async stopJob(executionId: number): Promise<void> {
    await api.post(`/batch/execution/stop/${executionId}`);
  }

  // Status & History
  async getExecutionStatus(executionId: number): Promise<BatchExecution> {
    const response = await api.get(`/batch/status/${executionId}`);
    return response.data;
  }

  async getExecutionHistory(params?: {
    page?: number;
    size?: number;
    jobConfigId?: number;
    status?: string;
    startDate?: string;
    endDate?: string;
  }): Promise<PaginatedResponse<BatchExecution>> {
    const response = await api.get('/batch/history', { params });
    return response.data;
  }

  async getRunningJobs(): Promise<BatchExecution[]> {
    const response = await api.get('/batch/status/running');
    return response.data;
  }

  async getDashboard(): Promise<{
    totalJobs: number;
    activeJobs: number;
    runningJobs: number;
    todayExecutions: number;
    successRate: number;
  }> {
    const response = await api.get('/batch/status/dashboard');
    return response.data;
  }

  // Table metadata
  async getTableNames(schemaName: string): Promise<string[]> {
    const response = await api.get(`/batch/metadata/tables`, {
      params: { schemaName },
    });
    return response.data;
  }

  async getTableColumns(schemaName: string, tableName: string): Promise<{
    columnName: string;
    dataType: string;
    nullable: boolean;
  }[]> {
    const response = await api.get(`/batch/metadata/columns`, {
      params: { schemaName, tableName },
    });
    return response.data;
  }
}

export const batchService = new BatchService();
```

**이유**: 클래스 기반 서비스로 관련 API 그룹화, 타입 안전성 확보, axios 인터셉터로 공통 처리

## 4. 타입 정의

### 4.1 batch.ts
**파일 위치**: `/frontend/src/types/batch.ts`
```typescript
export interface BatchJobConfig {
  jobConfigId: number;
  jobName: string;
  schemaName: string;
  tableName: string;
  startDate?: string;
  endDate?: string;
  insertQuery?: string;
  memo?: string;
  isActive: boolean;
  createdBy: string;
  createdDate: string;
  modifiedBy?: string;
  modifiedDate?: string;
}

export interface BatchExecution {
  executionId: number;
  jobConfig: BatchJobConfig;
  jobInstanceId?: number;
  jobExecutionId?: number;
  executionStatus: ExecutionStatus;
  startTime?: string;
  endTime?: string;
  durationSeconds?: number;
  totalCount: number;
  successCount: number;
  errorCount: number;
  skipCount: number;
  errorMessage?: string;
  executedBy?: string;
}

export type ExecutionStatus = 'STARTED' | 'COMPLETED' | 'FAILED' | 'STOPPED' | 'ABANDONED';

export interface PaginatedResponse<T> {
  content: T[];
  pageable: {
    pageNumber: number;
    pageSize: number;
    sort: {
      sorted: boolean;
      unsorted: boolean;
    };
  };
  totalElements: number;
  totalPages: number;
  size: number;
  number: number;
  first: boolean;
  last: boolean;
}

export interface TableColumn {
  columnName: string;
  dataType: string;
  typeName: string;
  columnSize: number;
  nullable: boolean;
  defaultValue?: string;
}

export interface DashboardStats {
  totalJobs: number;
  activeJobs: number;
  runningJobs: number;
  todayExecutions: number;
  successRate: number;
  recentExecutions: BatchExecution[];
}
```

**이유**: 백엔드 API 응답과 일치하는 타입 정의, 재사용 가능한 제네릭 타입

## 5. 공통 컴포넌트

### 5.1 Layout.tsx
**파일 위치**: `/frontend/src/components/Layout.tsx`
```tsx
import React, { useState } from 'react';
import {
  Box,
  Drawer,
  AppBar,
  Toolbar,
  List,
  Typography,
  Divider,
  IconButton,
  ListItem,
  ListItemButton,
  ListItemIcon,
  ListItemText,
  useTheme,
  useMediaQuery,
} from '@mui/material';
import {
  Menu as MenuIcon,
  Dashboard as DashboardIcon,
  Work as WorkIcon,
  History as HistoryIcon,
  Schedule as ScheduleIcon,
  Settings as SettingsIcon,
  Brightness4,
  Brightness7,
} from '@mui/icons-material';
import { useLocation, useNavigate } from 'react-router-dom';
import { useThemeStore } from '@/stores/themeStore';

const drawerWidth = 240;

interface LayoutProps {
  children: React.ReactNode;
}

const menuItems = [
  { path: '/', icon: <DashboardIcon />, label: '대시보드' },
  { path: '/jobs', icon: <WorkIcon />, label: '배치 작업 관리' },
  { path: '/history', icon: <HistoryIcon />, label: '실행 히스토리' },
  { path: '/schedule', icon: <ScheduleIcon />, label: '스케줄 관리' },
  { path: '/settings', icon: <SettingsIcon />, label: '설정' },
];

export default function Layout({ children }: LayoutProps) {
  const theme = useTheme();
  const isMobile = useMediaQuery(theme.breakpoints.down('md'));
  const [mobileOpen, setMobileOpen] = useState(false);
  const location = useLocation();
  const navigate = useNavigate();
  const { darkMode, toggleDarkMode } = useThemeStore();

  const handleDrawerToggle = () => {
    setMobileOpen(!mobileOpen);
  };

  const handleNavigation = (path: string) => {
    navigate(path);
    if (isMobile) {
      setMobileOpen(false);
    }
  };

  const drawer = (
    <div>
      <Toolbar>
        <Typography variant="h6" noWrap component="div">
          SBS DataSync
        </Typography>
      </Toolbar>
      <Divider />
      <List>
        {menuItems.map((item) => (
          <ListItem key={item.path} disablePadding>
            <ListItemButton
              selected={location.pathname === item.path}
              onClick={() => handleNavigation(item.path)}
            >
              <ListItemIcon>{item.icon}</ListItemIcon>
              <ListItemText primary={item.label} />
            </ListItemButton>
          </ListItem>
        ))}
      </List>
    </div>
  );

  return (
    <Box sx={{ display: 'flex' }}>
      <AppBar
        position="fixed"
        sx={{
          width: { md: `calc(100% - ${drawerWidth}px)` },
          ml: { md: `${drawerWidth}px` },
        }}
      >
        <Toolbar>
          <IconButton
            color="inherit"
            aria-label="open drawer"
            edge="start"
            onClick={handleDrawerToggle}
            sx={{ mr: 2, display: { md: 'none' } }}
          >
            <MenuIcon />
          </IconButton>
          <Typography variant="h6" noWrap component="div" sx={{ flexGrow: 1 }}>
            {menuItems.find(item => item.path === location.pathname)?.label || '대시보드'}
          </Typography>
          <IconButton color="inherit" onClick={toggleDarkMode}>
            {darkMode ? <Brightness7 /> : <Brightness4 />}
          </IconButton>
        </Toolbar>
      </AppBar>

      <Box
        component="nav"
        sx={{ width: { md: drawerWidth }, flexShrink: { md: 0 } }}
        aria-label="mailbox folders"
      >
        <Drawer
          variant="temporary"
          open={mobileOpen}
          onClose={handleDrawerToggle}
          ModalProps={{
            keepMounted: true,
          }}
          sx={{
            display: { xs: 'block', md: 'none' },
            '& .MuiDrawer-paper': { boxSizing: 'border-box', width: drawerWidth },
          }}
        >
          {drawer}
        </Drawer>
        <Drawer
          variant="permanent"
          sx={{
            display: { xs: 'none', md: 'block' },
            '& .MuiDrawer-paper': { boxSizing: 'border-box', width: drawerWidth },
          }}
          open
        >
          {drawer}
        </Drawer>
      </Box>

      <Box
        component="main"
        sx={{
          flexGrow: 1,
          p: 3,
          width: { md: `calc(100% - ${drawerWidth}px)` },
        }}
      >
        <Toolbar />
        {children}
      </Box>
    </Box>
  );
}
```

**이유**: Material-UI의 반응형 레이아웃, 다크 모드 토글, 네비게이션 상태 관리

### 5.2 LoadingSpinner.tsx
**파일 위치**: `/frontend/src/components/LoadingSpinner.tsx`
```tsx
import React from 'react';
import { Box, CircularProgress, Typography } from '@mui/material';

interface LoadingSpinnerProps {
  message?: string;
  size?: number;
}

export default function LoadingSpinner({
  message = '로딩 중...',
  size = 40
}: LoadingSpinnerProps) {
  return (
    <Box
      display="flex"
      flexDirection="column"
      alignItems="center"
      justifyContent="center"
      minHeight="200px"
      gap={2}
    >
      <CircularProgress size={size} />
      <Typography variant="body2" color="textSecondary">
        {message}
      </Typography>
    </Box>
  );
}
```

### 5.3 ErrorBoundary.tsx
**파일 위치**: `/frontend/src/components/ErrorBoundary.tsx`
```tsx
import React from 'react';
import { Box, Typography, Button, Paper } from '@mui/material';
import { ErrorOutline } from '@mui/icons-material';

interface ErrorBoundaryState {
  hasError: boolean;
  error?: Error;
}

class ErrorBoundary extends React.Component<
  React.PropsWithChildren<{}>,
  ErrorBoundaryState
> {
  constructor(props: React.PropsWithChildren<{}>) {
    super(props);
    this.state = { hasError: false };
  }

  static getDerivedStateFromError(error: Error): ErrorBoundaryState {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, errorInfo: React.ErrorInfo) {
    console.error('ErrorBoundary caught an error:', error, errorInfo);
  }

  render() {
    if (this.state.hasError) {
      return (
        <Box
          display="flex"
          flexDirection="column"
          alignItems="center"
          justifyContent="center"
          minHeight="400px"
          p={3}
        >
          <Paper
            elevation={3}
            sx={{
              p: 4,
              textAlign: 'center',
              maxWidth: 500,
            }}
          >
            <ErrorOutline
              sx={{
                fontSize: 64,
                color: 'error.main',
                mb: 2,
              }}
            />
            <Typography variant="h5" gutterBottom>
              오류가 발생했습니다
            </Typography>
            <Typography variant="body1" color="textSecondary" gutterBottom>
              예상치 못한 오류가 발생했습니다. 페이지를 새로고침하거나 관리자에게 문의하세요.
            </Typography>
            {this.state.error && (
              <Typography
                variant="caption"
                component="pre"
                sx={{
                  mt: 2,
                  p: 2,
                  bgcolor: 'grey.100',
                  borderRadius: 1,
                  fontSize: '0.75rem',
                  overflow: 'auto',
                  maxHeight: 200,
                }}
              >
                {this.state.error.message}
              </Typography>
            )}
            <Button
              variant="contained"
              onClick={() => window.location.reload()}
              sx={{ mt: 3 }}
            >
              페이지 새로고침
            </Button>
          </Paper>
        </Box>
      );
    }

    return this.props.children;
  }
}

export default ErrorBoundary;
```

**이유**: React 에러 경계로 예상치 못한 오류 처리, 사용자 친화적 에러 UI 제공

## 6. 기본 페이지 컴포넌트

### 6.1 Dashboard.tsx
**파일 위치**: `/frontend/src/pages/Dashboard.tsx`
```tsx
import React from 'react';
import {
  Grid,
  Card,
  CardContent,
  Typography,
  Box,
  LinearProgress,
} from '@mui/material';
import {
  Work as WorkIcon,
  PlayArrow as PlayIcon,
  CheckCircle as CheckIcon,
  Error as ErrorIcon,
} from '@mui/icons-material';
import { useQuery } from '@tanstack/react-query';
import { batchService } from '@/services/batchService';
import LoadingSpinner from '@/components/LoadingSpinner';

interface StatCardProps {
  title: string;
  value: number;
  icon: React.ReactNode;
  color?: string;
}

function StatCard({ title, value, icon, color = 'primary.main' }: StatCardProps) {
  return (
    <Card>
      <CardContent>
        <Box display="flex" alignItems="center" justifyContent="space-between">
          <Box>
            <Typography color="textSecondary" gutterBottom variant="h6">
              {title}
            </Typography>
            <Typography variant="h4" component="div">
              {value.toLocaleString()}
            </Typography>
          </Box>
          <Box sx={{ color, opacity: 0.7 }}>
            {icon}
          </Box>
        </Box>
      </CardContent>
    </Card>
  );
}

export default function Dashboard() {
  const { data: dashboardData, isLoading, error } = useQuery({
    queryKey: ['dashboard'],
    queryFn: () => batchService.getDashboard(),
    refetchInterval: 30000, // 30초마다 갱신
  });

  if (isLoading) {
    return <LoadingSpinner message="대시보드 데이터를 불러오는 중..." />;
  }

  if (error) {
    return (
      <Box textAlign="center" py={4}>
        <Typography variant="h6" color="error">
          대시보드 데이터를 불러오는데 실패했습니다.
        </Typography>
      </Box>
    );
  }

  return (
    <Box>
      <Typography variant="h4" gutterBottom>
        대시보드
      </Typography>

      <Grid container spacing={3} sx={{ mb: 4 }}>
        <Grid item xs={12} sm={6} md={3}>
          <StatCard
            title="전체 작업"
            value={dashboardData?.totalJobs || 0}
            icon={<WorkIcon sx={{ fontSize: 40 }} />}
            color="primary.main"
          />
        </Grid>
        <Grid item xs={12} sm={6} md={3}>
          <StatCard
            title="활성 작업"
            value={dashboardData?.activeJobs || 0}
            icon={<CheckIcon sx={{ fontSize: 40 }} />}
            color="success.main"
          />
        </Grid>
        <Grid item xs={12} sm={6} md={3}>
          <StatCard
            title="실행 중"
            value={dashboardData?.runningJobs || 0}
            icon={<PlayIcon sx={{ fontSize: 40 }} />}
            color="info.main"
          />
        </Grid>
        <Grid item xs={12} sm={6} md={3}>
          <StatCard
            title="오늘 실행"
            value={dashboardData?.todayExecutions || 0}
            icon={<PlayIcon sx={{ fontSize: 40 }} />}
            color="warning.main"
          />
        </Grid>
      </Grid>

      <Grid container spacing={3}>
        <Grid item xs={12} md={6}>
          <Card>
            <CardContent>
              <Typography variant="h6" gutterBottom>
                성공률
              </Typography>
              <Typography variant="h3" color="success.main" gutterBottom>
                {dashboardData?.successRate || 0}%
              </Typography>
              <LinearProgress
                variant="determinate"
                value={dashboardData?.successRate || 0}
                sx={{ height: 8, borderRadius: 4 }}
              />
            </CardContent>
          </Card>
        </Grid>
        <Grid item xs={12} md={6}>
          <Card>
            <CardContent>
              <Typography variant="h6" gutterBottom>
                시스템 상태
              </Typography>
              <Typography variant="body1" color="success.main">
                정상 운영 중
              </Typography>
              <Typography variant="body2" color="textSecondary">
                마지막 업데이트: {new Date().toLocaleString()}
              </Typography>
            </CardContent>
          </Card>
        </Grid>
      </Grid>
    </Box>
  );
}
```

**이유**: React Query로 실시간 데이터 페칭, Material-UI 카드 레이아웃, 통계 정보 시각화

### 6.2 BatchJobList.tsx (기본 구조)
**파일 위치**: `/frontend/src/pages/BatchJobList.tsx`
```tsx
import React, { useState } from 'react';
import {
  Box,
  Typography,
  Button,
  Card,
  CardContent,
  Fab,
} from '@mui/material';
import { Add as AddIcon } from '@mui/icons-material';
import { useQuery } from '@tanstack/react-query';
import { batchService } from '@/services/batchService';
import LoadingSpinner from '@/components/LoadingSpinner';

export default function BatchJobList() {
  const [page, setPage] = useState(0);
  const [pageSize, setPageSize] = useState(10);

  const { data: jobsData, isLoading } = useQuery({
    queryKey: ['batch-jobs', page, pageSize],
    queryFn: () => batchService.getJobs({ page, size: pageSize }),
  });

  if (isLoading) {
    return <LoadingSpinner message="배치 작업 목록을 불러오는 중..." />;
  }

  return (
    <Box>
      <Box display="flex" justifyContent="space-between" alignItems="center" mb={3}>
        <Typography variant="h4">배치 작업 관리</Typography>
        <Button
          variant="contained"
          startIcon={<AddIcon />}
          onClick={() => {/* TODO: 작업 추가 모달 */}}
        >
          새 작업 추가
        </Button>
      </Box>

      <Card>
        <CardContent>
          <Typography variant="h6">
            총 {jobsData?.totalElements || 0}개의 작업
          </Typography>
          {/* TODO: 데이터 그리드 구현 */}
        </CardContent>
      </Card>
    </Box>
  );
}
```

이상으로 Phase 3의 주요 구현 코드를 제시했습니다. 이 단계에서는 React 기본 구조, 상태 관리, API 연동, 공통 컴포넌트의 기초를 마련합니다.