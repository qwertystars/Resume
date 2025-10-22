# Dashboards & UI Implementation Guide

## Overview
This guide covers building the complete frontend UI including candidate dashboard, data tables, filters, search, and all interactive components.

---

## Tech Stack

- **React 18** with TypeScript
- **Vite** for build tooling
- **Tailwind CSS** for styling
- **Shadcn/ui** for base components
- **TanStack Table** (React Table v8) for data tables
- **Recharts** for analytics charts
- **React Router** for navigation
- **Zustand** for state management
- **React Query** for server state

---

## Project Setup

### 1. Initialize Project

```bash
npm create vite@latest hiresight-frontend -- --template react-ts
cd hiresight-frontend
npm install

# Install dependencies
npm install react-router-dom zustand @tanstack/react-query axios
npm install recharts lucide-react react-hook-form zod
npm install @tanstack/react-table react-dropzone date-fns

# Install Tailwind
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p
```

### 2. Tailwind Configuration

**File: `tailwind.config.js`**

```javascript
export default {
  content: [
    "./index.html",
    "./src/**/*.{js,ts,jsx,tsx}",
  ],
  theme: {
    extend: {
      colors: {
        primary: {
          50: '#eff6ff',
          500: '#3b82f6',
          600: '#2563eb',
          700: '#1d4ed8',
        }
      }
    },
  },
  plugins: [],
}
```

---

## Core Components

### 1. Candidate Dashboard

**File: `frontend/src/pages/CandidatesPage.tsx`**

```typescript
import React, { useState } from 'react';
import { useQuery } from '@tanstack/react-query';
import { CandidateTable } from '../components/candidates/CandidateTable';
import { CandidateFilters } from '../components/candidates/CandidateFilters';
import { SearchBar } from '../components/common/SearchBar';
import { Button } from '../components/ui/Button';
import { Plus, Upload } from 'lucide-react';
import { getCandidates } from '../services/candidateService';

export const CandidatesPage: React.FC = () => {
  const [filters, setFilters] = useState({});
  const [search, setSearch] = useState('');
  const [page, setPage] = useState(1);

  const { data, isLoading, error } = useQuery({
    queryKey: ['candidates', filters, search, page],
    queryFn: () => getCandidates({ ...filters, search, page, per_page: 20 })
  });

  return (
    <div className="min-h-screen bg-gray-50">
      {/* Header */}
      <div className="bg-white border-b">
        <div className="max-w-7xl mx-auto px-4 py-6">
          <div className="flex items-center justify-between">
            <div>
              <h1 className="text-2xl font-bold text-gray-900">Candidates</h1>
              <p className="text-gray-600 mt-1">
                {data?.metadata.total || 0} total candidates
              </p>
            </div>

            <div className="flex items-center space-x-3">
              <Button variant="outline" icon={<Upload />}>
                Import Resumes
              </Button>
              <Button icon={<Plus />}>
                Add Candidate
              </Button>
            </div>
          </div>

          {/* Search */}
          <div className="mt-6">
            <SearchBar
              placeholder="Search candidates..."
              value={search}
              onChange={setSearch}
            />
          </div>
        </div>
      </div>

      {/* Main Content */}
      <div className="max-w-7xl mx-auto px-4 py-6">
        <div className="flex gap-6">
          {/* Filters Sidebar */}
          <aside className="w-64 flex-shrink-0">
            <CandidateFilters
              filters={filters}
              onChange={setFilters}
            />
          </aside>

          {/* Table */}
          <main className="flex-1">
            {isLoading ? (
              <div className="flex items-center justify-center h-64">
                <div className="animate-spin rounded-full h-12 w-12 border-b-2 border-blue-600" />
              </div>
            ) : error ? (
              <div className="text-center text-red-600 p-4">
                Error loading candidates
              </div>
            ) : (
              <CandidateTable
                data={data.data.candidates}
                pagination={data.metadata}
                onPageChange={setPage}
              />
            )}
          </main>
        </div>
      </div>
    </div>
  );
};
```

### 2. Candidate Data Table

**File: `frontend/src/components/candidates/CandidateTable.tsx`**

```typescript
import React from 'react';
import {
  useReactTable,
  getCoreRowModel,
  getSortedRowModel,
  flexRender,
  ColumnDef,
  SortingState,
} from '@tanstack/react-table';
import { Candidate } from '../../types';
import { ChevronUp, ChevronDown, Star } from 'lucide-react';
import { useNavigate } from 'react-router-dom';

interface CandidateTableProps {
  data: Candidate[];
  pagination: any;
  onPageChange: (page: number) => void;
}

export const CandidateTable: React.FC<CandidateTableProps> = ({
  data,
  pagination,
  onPageChange
}) => {
  const navigate = useNavigate();
  const [sorting, setSorting] = React.useState<SortingState>([]);

  const columns: ColumnDef<Candidate>[] = [
    {
      accessorKey: 'full_name',
      header: 'Name',
      cell: ({ row }) => (
        <div className="flex items-center space-x-3">
          <div className="w-10 h-10 bg-blue-100 rounded-full flex items-center justify-center">
            <span className="text-blue-600 font-semibold">
              {row.original.full_name.charAt(0)}
            </span>
          </div>
          <div>
            <div className="font-medium text-gray-900">
              {row.original.full_name}
            </div>
            <div className="text-sm text-gray-500">
              {row.original.email}
            </div>
          </div>
        </div>
      ),
    },
    {
      accessorKey: 'status',
      header: 'Status',
      cell: ({ row }) => (
        <span className={`inline-flex items-center px-2.5 py-0.5 rounded-full text-xs font-medium
          ${row.original.status === 'hired' ? 'bg-green-100 text-green-800' :
            row.original.status === 'interviewing' ? 'bg-blue-100 text-blue-800' :
            row.original.status === 'rejected' ? 'bg-red-100 text-red-800' :
            'bg-gray-100 text-gray-800'}`}>
          {row.original.status}
        </span>
      ),
    },
    {
      accessorKey: 'overall_score',
      header: 'Score',
      cell: ({ row }) => (
        <div className="flex items-center space-x-2">
          <div className="flex items-center">
            <Star className="w-4 h-4 text-yellow-400 fill-current" />
            <span className="ml-1 font-semibold text-gray-900">
              {row.original.overall_score}
            </span>
          </div>
          <div className="w-20 h-2 bg-gray-200 rounded-full overflow-hidden">
            <div
              className={`h-full ${
                row.original.overall_score >= 80 ? 'bg-green-500' :
                row.original.overall_score >= 60 ? 'bg-yellow-500' :
                'bg-red-500'
              }`}
              style={{ width: `${row.original.overall_score}%` }}
            />
          </div>
        </div>
      ),
    },
    {
      accessorKey: 'years_of_experience',
      header: 'Experience',
      cell: ({ row }) => (
        <span className="text-gray-700">
          {row.original.years_of_experience} years
        </span>
      ),
    },
    {
      accessorKey: 'primary_skills',
      header: 'Skills',
      cell: ({ row }) => (
        <div className="flex flex-wrap gap-1">
          {row.original.primary_skills?.slice(0, 3).map((skill) => (
            <span
              key={skill}
              className="inline-flex items-center px-2 py-0.5 rounded text-xs bg-blue-50 text-blue-700"
            >
              {skill}
            </span>
          ))}
          {row.original.primary_skills?.length > 3 && (
            <span className="text-xs text-gray-500">
              +{row.original.primary_skills.length - 3}
            </span>
          )}
        </div>
      ),
    },
    {
      accessorKey: 'location',
      header: 'Location',
      cell: ({ row }) => (
        <span className="text-gray-700">{row.original.location}</span>
      ),
    },
  ];

  const table = useReactTable({
    data,
    columns,
    state: { sorting },
    onSortingChange: setSorting,
    getCoreRowModel: getCoreRowModel(),
    getSortedRowModel: getSortedRowModel(),
  });

  return (
    <div className="bg-white rounded-lg shadow">
      <div className="overflow-x-auto">
        <table className="min-w-full divide-y divide-gray-200">
          <thead className="bg-gray-50">
            {table.getHeaderGroups().map(headerGroup => (
              <tr key={headerGroup.id}>
                {headerGroup.headers.map(header => (
                  <th
                    key={header.id}
                    className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider cursor-pointer hover:bg-gray-100"
                    onClick={header.column.getToggleSortingHandler()}
                  >
                    <div className="flex items-center space-x-1">
                      <span>
                        {flexRender(header.column.columnDef.header, header.getContext())}
                      </span>
                      {header.column.getIsSorted() && (
                        header.column.getIsSorted() === 'asc' ?
                          <ChevronUp className="w-4 h-4" /> :
                          <ChevronDown className="w-4 h-4" />
                      )}
                    </div>
                  </th>
                ))}
              </tr>
            ))}
          </thead>
          <tbody className="bg-white divide-y divide-gray-200">
            {table.getRowModel().rows.map(row => (
              <tr
                key={row.id}
                className="hover:bg-gray-50 cursor-pointer"
                onClick={() => navigate(`/candidates/${row.original.id}`)}
              >
                {row.getVisibleCells().map(cell => (
                  <td key={cell.id} className="px-6 py-4 whitespace-nowrap">
                    {flexRender(cell.column.columnDef.cell, cell.getContext())}
                  </td>
                ))}
              </tr>
            ))}
          </tbody>
        </table>
      </div>

      {/* Pagination */}
      <div className="bg-white px-4 py-3 border-t border-gray-200 sm:px-6">
        <div className="flex items-center justify-between">
          <div className="text-sm text-gray-700">
            Showing <span className="font-medium">{(pagination.page - 1) * pagination.per_page + 1}</span> to{' '}
            <span className="font-medium">
              {Math.min(pagination.page * pagination.per_page, pagination.total)}
            </span> of{' '}
            <span className="font-medium">{pagination.total}</span> results
          </div>

          <div className="flex space-x-2">
            <button
              onClick={() => onPageChange(pagination.page - 1)}
              disabled={pagination.page === 1}
              className="px-3 py-1 border rounded disabled:opacity-50 disabled:cursor-not-allowed"
            >
              Previous
            </button>
            <button
              onClick={() => onPageChange(pagination.page + 1)}
              disabled={pagination.page * pagination.per_page >= pagination.total}
              className="px-3 py-1 border rounded disabled:opacity-50 disabled:cursor-not-allowed"
            >
              Next
            </button>
          </div>
        </div>
      </div>
    </div>
  );
};
```

### 3. Filters Sidebar

**File: `frontend/src/components/candidates/CandidateFilters.tsx`**

```typescript
import React from 'react';
import { Filter, X } from 'lucide-react';

interface FilterProps {
  filters: any;
  onChange: (filters: any) => void;
}

export const CandidateFilters: React.FC<FilterProps> = ({ filters, onChange }) => {
  const updateFilter = (key: string, value: any) => {
    onChange({ ...filters, [key]: value });
  };

  const clearFilters = () => {
    onChange({});
  };

  return (
    <div className="bg-white rounded-lg shadow p-4">
      <div className="flex items-center justify-between mb-4">
        <div className="flex items-center space-x-2">
          <Filter className="w-5 h-5 text-gray-600" />
          <h3 className="font-semibold text-gray-900">Filters</h3>
        </div>
        {Object.keys(filters).length > 0 && (
          <button
            onClick={clearFilters}
            className="text-sm text-blue-600 hover:text-blue-800"
          >
            Clear all
          </button>
        )}
      </div>

      <div className="space-y-4">
        {/* Status Filter */}
        <div>
          <label className="block text-sm font-medium text-gray-700 mb-2">
            Status
          </label>
          <select
            value={filters.status || ''}
            onChange={(e) => updateFilter('status', e.target.value)}
            className="w-full border rounded-lg px-3 py-2 text-sm"
          >
            <option value="">All Statuses</option>
            <option value="new">New</option>
            <option value="reviewing">Reviewing</option>
            <option value="interviewing">Interviewing</option>
            <option value="offered">Offered</option>
            <option value="hired">Hired</option>
            <option value="rejected">Rejected</option>
          </select>
        </div>

        {/* Score Range */}
        <div>
          <label className="block text-sm font-medium text-gray-700 mb-2">
            Minimum Score
          </label>
          <input
            type="range"
            min="0"
            max="100"
            value={filters.min_score || 0}
            onChange={(e) => updateFilter('min_score', e.target.value)}
            className="w-full"
          />
          <div className="flex justify-between text-xs text-gray-500 mt-1">
            <span>0</span>
            <span className="font-semibold text-blue-600">
              {filters.min_score || 0}
            </span>
            <span>100</span>
          </div>
        </div>

        {/* Experience */}
        <div>
          <label className="block text-sm font-medium text-gray-700 mb-2">
            Experience (years)
          </label>
          <div className="flex space-x-2">
            <input
              type="number"
              placeholder="Min"
              value={filters.min_experience || ''}
              onChange={(e) => updateFilter('min_experience', e.target.value)}
              className="w-1/2 border rounded-lg px-3 py-2 text-sm"
            />
            <input
              type="number"
              placeholder="Max"
              value={filters.max_experience || ''}
              onChange={(e) => updateFilter('max_experience', e.target.value)}
              className="w-1/2 border rounded-lg px-3 py-2 text-sm"
            />
          </div>
        </div>

        {/* Location */}
        <div>
          <label className="block text-sm font-medium text-gray-700 mb-2">
            Location
          </label>
          <input
            type="text"
            placeholder="City, State"
            value={filters.location || ''}
            onChange={(e) => updateFilter('location', e.target.value)}
            className="w-full border rounded-lg px-3 py-2 text-sm"
          />
        </div>

        {/* Skills */}
        <div>
          <label className="block text-sm font-medium text-gray-700 mb-2">
            Skills
          </label>
          <input
            type="text"
            placeholder="Python, React, etc."
            value={filters.skills || ''}
            onChange={(e) => updateFilter('skills', e.target.value)}
            className="w-full border rounded-lg px-3 py-2 text-sm"
          />
          <p className="text-xs text-gray-500 mt-1">
            Comma-separated
          </p>
        </div>

        {/* Education */}
        <div>
          <label className="block text-sm font-medium text-gray-700 mb-2">
            Education
          </label>
          <select
            value={filters.education_level || ''}
            onChange={(e) => updateFilter('education_level', e.target.value)}
            className="w-full border rounded-lg px-3 py-2 text-sm"
          >
            <option value="">Any</option>
            <option value="high_school">High School</option>
            <option value="associate">Associate</option>
            <option value="bachelor">Bachelor</option>
            <option value="master">Master</option>
            <option value="phd">PhD</option>
          </select>
        </div>

        {/* Source */}
        <div>
          <label className="block text-sm font-medium text-gray-700 mb-2">
            Source
          </label>
          <select
            value={filters.source || ''}
            onChange={(e) => updateFilter('source', e.target.value)}
            className="w-full border rounded-lg px-3 py-2 text-sm"
          >
            <option value="">All Sources</option>
            <option value="linkedin">LinkedIn</option>
            <option value="job_board">Job Board</option>
            <option value="referral">Referral</option>
            <option value="direct_apply">Direct Apply</option>
          </select>
        </div>
      </div>
    </div>
  );
};
```

### 4. Candidate Detail Page

**File: `frontend/src/pages/CandidateDetailPage.tsx`**

```typescript
import React from 'react';
import { useParams } from 'react-router-dom';
import { useQuery } from '@tanstack/react-query';
import { getCandidate } from '../services/candidateService';
import { ScoreDisplay } from '../components/scoring/ScoreDisplay';
import { GitHubProfile } from '../components/github/GitHubProfile';
import { ActivityTimeline } from '../components/candidates/ActivityTimeline';
import { Mail, Phone, MapPin, Linkedin, Github, Globe } from 'lucide-react';

export const CandidateDetailPage: React.FC = () => {
  const { id } = useParams<{ id: string }>();

  const { data, isLoading } = useQuery({
    queryKey: ['candidate', id],
    queryFn: () => getCandidate(id!)
  });

  if (isLoading) {
    return <div>Loading...</div>;
  }

  const candidate = data?.data.candidate;

  return (
    <div className="min-h-screen bg-gray-50">
      {/* Header */}
      <div className="bg-white border-b">
        <div className="max-w-7xl mx-auto px-4 py-6">
          <div className="flex items-start space-x-4">
            <div className="w-20 h-20 bg-blue-100 rounded-full flex items-center justify-center">
              <span className="text-3xl text-blue-600 font-semibold">
                {candidate.full_name.charAt(0)}
              </span>
            </div>

            <div className="flex-1">
              <h1 className="text-3xl font-bold text-gray-900">
                {candidate.full_name}
              </h1>

              <div className="flex items-center space-x-4 mt-2 text-gray-600">
                {candidate.email && (
                  <a href={`mailto:${candidate.email}`} className="flex items-center space-x-1 hover:text-blue-600">
                    <Mail className="w-4 h-4" />
                    <span>{candidate.email}</span>
                  </a>
                )}
                {candidate.phone && (
                  <div className="flex items-center space-x-1">
                    <Phone className="w-4 h-4" />
                    <span>{candidate.phone}</span>
                  </div>
                )}
                {candidate.location && (
                  <div className="flex items-center space-x-1">
                    <MapPin className="w-4 h-4" />
                    <span>{candidate.location}</span>
                  </div>
                )}
              </div>

              <div className="flex items-center space-x-3 mt-3">
                {candidate.linkedin_url && (
                  <a href={candidate.linkedin_url} target="_blank" className="text-blue-600 hover:underline">
                    <Linkedin className="w-5 h-5" />
                  </a>
                )}
                {candidate.github_url && (
                  <a href={candidate.github_url} target="_blank" className="text-gray-700 hover:text-gray-900">
                    <Github className="w-5 h-5" />
                  </a>
                )}
                {candidate.portfolio_url && (
                  <a href={candidate.portfolio_url} target="_blank" className="text-gray-700 hover:text-gray-900">
                    <Globe className="w-5 h-5" />
                  </a>
                )}
              </div>
            </div>
          </div>
        </div>
      </div>

      {/* Content */}
      <div className="max-w-7xl mx-auto px-4 py-6">
        <div className="grid grid-cols-3 gap-6">
          {/* Main Content */}
          <div className="col-span-2 space-y-6">
            {/* Skills */}
            <div className="bg-white rounded-lg shadow p-6">
              <h3 className="text-lg font-semibold mb-4">Skills</h3>
              <div className="flex flex-wrap gap-2">
                {candidate.primary_skills?.map((skill: string) => (
                  <span key={skill} className="px-3 py-1 bg-blue-50 text-blue-700 rounded-lg text-sm font-medium">
                    {skill}
                  </span>
                ))}
              </div>
            </div>

            {/* GitHub Profile */}
            {candidate.github_profile && (
              <GitHubProfile profile={candidate.github_profile} />
            )}

            {/* Activity Timeline */}
            <ActivityTimeline activities={candidate.activities || []} />
          </div>

          {/* Sidebar */}
          <div className="space-y-6">
            {/* Score */}
            <ScoreDisplay
              overall={candidate.overall_score}
              breakdown={{
                resume: candidate.resume_score,
                github: candidate.github_score,
                linkedin: candidate.linkedin_score,
                skills: 0,
                experience: 0,
                education: 0
              }}
              percentile={75}
            />

            {/* Quick Info */}
            <div className="bg-white rounded-lg shadow p-6">
              <h3 className="font-semibold mb-4">Quick Info</h3>
              <dl className="space-y-3">
                <div>
                  <dt className="text-sm text-gray-500">Status</dt>
                  <dd className="mt-1 font-medium">{candidate.status}</dd>
                </div>
                <div>
                  <dt className="text-sm text-gray-500">Experience</dt>
                  <dd className="mt-1 font-medium">{candidate.years_of_experience} years</dd>
                </div>
                <div>
                  <dt className="text-sm text-gray-500">Education</dt>
                  <dd className="mt-1 font-medium capitalize">{candidate.education_level}</dd>
                </div>
                <div>
                  <dt className="text-sm text-gray-500">Source</dt>
                  <dd className="mt-1 font-medium capitalize">{candidate.source}</dd>
                </div>
              </dl>
            </div>
          </div>
        </div>
      </div>
    </div>
  );
};
```

---

## State Management with Zustand

**File: `frontend/src/stores/uiStore.ts`**

```typescript
import { create } from 'zustand';

interface UIState {
  sidebarOpen: boolean;
  toggleSidebar: () => void;
  theme: 'light' | 'dark';
  setTheme: (theme: 'light' | 'dark') => void;
}

export const useUIStore = create<UIState>((set) => ({
  sidebarOpen: true,
  toggleSidebar: () => set((state) => ({ sidebarOpen: !state.sidebarOpen })),
  theme: 'light',
  setTheme: (theme) => set({ theme }),
}));
```

---

## API Client Setup

**File: `frontend/src/services/api.ts`**

```typescript
import axios from 'axios';

export const api = axios.create({
  baseURL: import.meta.env.VITE_API_URL || 'http://localhost:8000/api/v1',
  headers: {
    'Content-Type': 'application/json',
  },
});

// Request interceptor
api.interceptors.request.use(
  (config) => {
    const token = localStorage.getItem('access_token');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
);

// Response interceptor
api.interceptors.response.use(
  (response) => response,
  async (error) => {
    const originalRequest = error.config;

    // Handle 401 errors
    if (error.response?.status === 401 && !originalRequest._retry) {
      originalRequest._retry = true;

      try {
        // Refresh token
        const refreshToken = localStorage.getItem('refresh_token');
        const response = await axios.post('/api/v1/auth/refresh', {
          refresh_token: refreshToken,
        });

        const { access_token } = response.data;
        localStorage.setItem('access_token', access_token);

        // Retry original request
        originalRequest.headers.Authorization = `Bearer ${access_token}`;
        return api(originalRequest);
      } catch (refreshError) {
        // Refresh failed, logout user
        localStorage.removeItem('access_token');
        localStorage.removeItem('refresh_token');
        window.location.href = '/login';
        return Promise.reject(refreshError);
      }
    }

    return Promise.reject(error);
  }
);
```

This comprehensive UI implementation provides a production-ready dashboard with all essential features for candidate management.
