# ESG Database Schema Assessment

> This is a well-designed, enterprise-grade schema that follows most best practices. The modular structure is particularly strong for team collaboration and maintenance.

---

## ✅ Strengths

### 1. Excellent Modular Architecture
- ✅ Separating concerns into domain-specific modules (water, GHG, food safety) is smart for large teams
- ✅ Clear dependency hierarchy with core schema first
- ✅ Each module can be developed/tested independently

### 2. Strong Normalization (Mostly 3NF+)
- ✅ Tables are well-normalized: No obvious repeating groups or transitive dependencies
- ✅ Proper key usage: UUID primary keys with consistent foreign key relationships
- ✅ Atomic data: Fields contain single values (not comma-separated lists)

### 3. Good Data Integrity Enforcement
- ✅ Foreign key constraints everywhere with proper ON DELETE actions
- ✅ NOT NULL constraints on essential fields
- ✅ Data types are appropriate (NUMERIC for calculations, TEXT for descriptions)

### 4. Consistent Naming Conventions
- ✅ snake_case throughout
- ✅ Clear table/field names aligned with ESG terminology
- ✅ Plural table names (organizations, not organization)

### 5. Future-Proofing Considered
- ✅ UUIDs support distributed systems
- ✅ Extensible design for new regulations (CSRD, ISO)
- ✅ Metadata fields (created_at, evidence_uri) support auditing

---

## ⚠️ Issues & Recommendations

### 1. Normalization Issues

#### Problem: Redundant Calculated Fields

```sql
-- In scope1_emission_events
calculated_emissions_tco2e NUMERIC  -- Derived from activity_amount * emission_factor_kg_per_unit
```

**Recommendation:** Remove calculated fields that can be derived. Store source data only, calculate in views/queries.

#### Problem: Potential Denormalization in sites

```sql
water_stress_level TEXT,  -- This might change over time but is stored statically
water_stress_source TEXT
```

**Recommendation:** Consider a time-series table `site_water_stress_history` if this data changes per reporting period.

---

### 2. Missing Constraints & Data Quality

#### Problem: Limited CHECK Constraints

```sql
-- Example: No validation on percentages
revenue_share_pct NUMERIC(5,2)  -- Could be >100 or <0
scope_coverage_pct NUMERIC(5,2)
```

**Recommendation:**

```sql
revenue_share_pct NUMERIC(5,2) CHECK (revenue_share_pct BETWEEN 0 AND 100),
scope_coverage_pct NUMERIC(5,2) CHECK (scope_coverage_pct BETWEEN 0 AND 100)
```

#### Problem: No ENUM Types for Coded Fields

```sql
source_type TEXT,  -- What are valid values?
severity TEXT,     -- 'minor', 'major', 'critical'?
employee_type TEXT -- 'direct', 'contract', 'seasonal'?
```

**Recommendation:** Use PostgreSQL ENUM types or reference tables:

```sql
CREATE TYPE incident_severity AS ENUM ('minor', 'major', 'critical');
CREATE TYPE employee_category AS ENUM ('direct', 'contract', 'seasonal');
```

---

### 3. Performance Considerations

#### Problem: Missing Indexes

Only primary keys are indexed. Foreign keys need indexes for JOIN performance:

```sql
-- Example: These columns need indexes for performance
CREATE INDEX idx_metric_values_org_period ON metric_values(org_id, period_id);
CREATE INDEX idx_scope1_events_facility ON scope1_emission_events(facility_id);
```

#### Problem: Wide Tables

`scope1_emission_events` has 17 columns. Consider splitting if some columns are rarely used together.

---

### 4. Design Inconsistencies

#### Problem: Mixed NULL Handling

```sql
ON DELETE SET NULL  -- Used inconsistently
ON DELETE CASCADE   -- Sometimes cascade, sometimes set null
```

**Recommendation:** Establish clear rules:
- Use **CASCADE** for strong dependencies (audit → nonconformance)
- Use **SET NULL** for optional relationships (facility reference)

#### Problem: Incomplete Relationships

```sql
CREATE TABLE principal_crops (
    crop_id UUID NOT NULL REFERENCES crops(crop_id),
    org_id UUID NOT NULL REFERENCES organizations(org_id),  -- Redundant? Already in crops table
    period_id UUID NOT NULL  -- Good: time-bound principal status
);
```

The `org_id` here might be redundant since `crops` already has `org_id`.

---

### 5. Business Logic in Schema

#### Problem: Business Rules Embedded

```sql
meets_recall_definition BOOLEAN DEFAULT TRUE  -- Business logic in DB
count_in_rate BOOLEAN DEFAULT TRUE            -- Should this be in application layer?
```

**Recommendation:** Document these flags clearly. Consider if they should be calculated rather than stored.

---

## 🔴 Critical Missing Components

### 1. No Audit Trail Tables

For compliance reporting, you need change tracking:

```sql
CREATE TABLE schema_audit_log (
    log_id UUID PRIMARY KEY,
    table_name TEXT,
    record_id UUID,
    changed_by TEXT,
    changed_at TIMESTAMPTZ,
    old_values JSONB,
    new_values JSONB
);
```

### 2. No Data Validation Tables

Missing reference tables for valid values:

```sql
CREATE TABLE emission_source_types (
    code TEXT PRIMARY KEY,
    description TEXT,
    category TEXT
);

CREATE TABLE water_stress_levels (
    level TEXT PRIMARY KEY,
    min_score NUMERIC,
    max_score NUMERIC,
    description TEXT
);
```

### 3. No Composite Unique Constraints

```sql
-- Prevent duplicate metric values for same org/period/metric
CREATE UNIQUE INDEX uq_metric_values_unique 
ON metric_values(org_id, period_id, metric_code) 
WHERE is_primary_disclosure = TRUE;

-- Prevent duplicate reporting periods for same org
CREATE UNIQUE INDEX uq_reporting_period_org 
ON reporting_periods(org_id, year_label);
```

### 4. Missing Temporal Support

Some data needs history tracking:

```sql
-- Add valid_from/valid_to for time-sensitive data
ALTER TABLE sites ADD COLUMN valid_from DATE;
ALTER TABLE sites ADD COLUMN valid_to DATE;
```

---

## 📋 Specific Table Reviews

### `metric_values` Table Issue

```sql
value_number NUMERIC,
value_text TEXT,
unit_override TEXT  -- Overrides metric_definitions.unit
```

**Problem:** Polymorphic value storage. Consider separate tables for numeric vs text metrics.

### `narrative_responses` Redundancy

This table duplicates foreign keys from `metric_values`. Consider linking to `metric_values` instead:

```sql
narrative_response_id UUID REFERENCES metric_values(metric_value_id)
```

### `work_hours` Design Issue

```sql
total_hours_worked NUMERIC NOT NULL  -- For entire period?
```

**Problem:** No breakdown by facility or date range. Difficult to reconcile with incidents.

---

## 🎯 Recommended Actions

### Immediate (Before Production)

| Priority | Action | Impact |
|----------|--------|--------|
| 🔴 High | Add foreign key indexes on all FK columns | Performance |
| 🔴 High | Add CHECK constraints for percentages, positive values | Data Quality |
| 🔴 High | Create reference tables for coded fields | Consistency |
| 🔴 High | Add unique constraints to prevent duplicates | Data Integrity |

### Short-term (Next Release)

| Priority | Action | Impact |
|----------|--------|--------|
| 🟡 Medium | Remove calculated fields from base tables | Normalization |
| 🟡 Medium | Standardize ON DELETE behavior | Consistency |
| 🟡 Medium | Add audit logging tables | Compliance |
| 🟡 Medium | Create materialized views for complex calculations | Performance |

### Long-term

| Priority | Action | Impact |
|----------|--------|--------|
| 🟢 Low | Consider partitioning by period_id for large tables | Scalability |
| 🟢 Low | Add full-text search indexes on narrative fields | Search Performance |
| 🟢 Low | Implement row-level security for multi-tenant data | Security |
| 🟢 Low | Create validation triggers for complex business rules | Data Quality |

---

## 💡 Alternative Design Considerations

### Option A: Event Sourcing for Metrics

Instead of `metric_values` table, consider:

```sql
CREATE TABLE metric_events (
    event_id UUID PRIMARY KEY,
    metric_code TEXT,
    org_id UUID,
    period_id UUID,
    value JSONB,  -- Flexible value storage
    calculated_value NUMERIC,  -- Materialized calculation
    calculation_method TEXT
);
```

**Benefits:**
- Complete audit trail
- Time-travel queries
- Supports complex calculations

### Option B: Polymorphic Relationships

For tables linking to multiple entity types (facility OR site OR supplier):

```sql
CREATE TABLE evidence_documents (
    doc_id UUID PRIMARY KEY,
    entity_type TEXT,  -- 'facility', 'site', 'supplier'
    entity_id UUID,    -- Flexible reference
    document_type TEXT,
    evidence_uri TEXT
);
```

**Benefits:**
- Flexible relationships
- Reduces table proliferation
- Easier to extend

---

## 📊 Summary Matrix

| Category | Score | Status |
|----------|-------|--------|
| **Architecture** | 9/10 | ✅ Excellent |
| **Normalization** | 8/10 | ⚠️ Minor issues |
| **Data Integrity** | 8/10 | ⚠️ Missing constraints |
| **Performance** | 7/10 | ⚠️ Needs indexing |
| **Maintainability** | 9/10 | ✅ Excellent |
| **Compliance** | 7/10 | ⚠️ Missing audit trails |
| **Scalability** | 8/10 | ✅ Good foundation |

---

## 🎓 Conclusion

This is a professionally designed schema that shows deep understanding of both database principles and ESG reporting requirements. The modular approach is excellent for team scalability.

### Key Fixes Needed:

1. ✅ Add indexes on foreign keys (performance)
2. ✅ Implement CHECK constraints (data quality)
3. ✅ Create reference tables for coded values (consistency)
4. ✅ Remove redundant calculated fields (normalization)

### The schema successfully balances:

- ✅ **Regulatory compliance** (audit trails, evidence tracking)
- ⚠️ **Performance** (though needs indexing)
- ✅ **Maintainability** (clear structure, good documentation)
- ✅ **Flexibility** (supports multiple reporting frameworks)

---

## 📚 Additional Resources

- [PostgreSQL Best Practices](https://www.postgresql.org/docs/current/ddl-constraints.html)
- [Database Normalization Guide](https://en.wikipedia.org/wiki/Database_normalization)
- [ESG Reporting Standards](https://www.globalreporting.org/)

---

**Last Updated:** 2024  
**Schema Version:** 1.0  
**Assessment Date:** Current
