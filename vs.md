# 👨‍💼 Employee Management Lightning Web Component — Complete Guide

## Problem Statement
Develop an Employee Management Lightning Web Component (LWC) that allows users to add employee records with full validation:
- Employee Name: cannot be empty, minimum 3 characters
- Employee ID: must be > 0 and unique
- Salary: must be > 10,000 and < 500,000
- Email: must follow valid email format
- Department: must be selected from available list
- Joining Date: cannot be a future date

---

## Prerequisites
- Salesforce Developer Account (free at [developer.salesforce.com](https://developer.salesforce.com))
- Salesforce CLI installed (`sf` or `sfdx`)
- VS Code with **Salesforce Extension Pack** installed
- Basic knowledge of LWC, Apex, HTML, JavaScript

---

## Architecture Overview

```
Lightning Web Component (Frontend)
        ↓ (user fills form + validates)
Apex Controller (Backend)
        ↓ (checks uniqueness + saves)
Salesforce Custom Object (Employee__c)
        ↓ (stored in database)
    LWC Table (refreshed after save)
```

---

## FILES TO CREATE

| File | Type | Purpose |
|------|------|---------|
| `Employee__c` | Custom Object | Store employee records |
| `EmployeeController.cls` | Apex Class | Backend CRUD operations |
| `employeeManagement.html` | LWC Template | Frontend UI |
| `employeeManagement.js` | LWC JavaScript | Frontend logic + validation |
| `employeeManagement.css` | LWC CSS | Styling |
| `employeeManagement.js-meta.xml` | LWC Metadata | Component config |

---

## STEP 1 — Create Custom Object (Employee__c)

### Navigate to:
**Setup → Object Manager → Create → Custom Object**

Fill in:
- **Label:** `Employee`
- **Plural Label:** `Employees`
- **Object Name:** `Employee__c`
- Check: **"Allow Reports"**, **"Allow Activities"**
- Click **Save**

### Add Custom Fields:
Go to **Object Manager → Employee__c → Fields & Relationships → New**

| Field Label | API Name | Type | Properties |
|-------------|----------|------|-----------|
| Employee Name | `Employee_Name__c` | Text | Length: 100, Required |
| Employee ID | `Employee_ID__c` | Number | Length: 10, Unique, Required |
| Salary | `Salary__c` | Currency | Length: 10, Decimal: 2, Required |
| Email | `Email__c` | Email | Required |
| Department | `Department__c` | Picklist | Values below, Required |
| Joining Date | `Joining_Date__c` | Date | Required |

### Department Picklist Values:
```
HR
Engineering
Finance
Marketing
Sales
Operations
IT
Legal
```

---

## STEP 2 — Create Apex Controller

### Navigate to:
**Setup → Developer Console → File → New → Apex Class**

Name: `EmployeeController`

### Code: `EmployeeController.cls`

```apex
public with sharing class EmployeeController {

    // Fetch all employees
    @AuraEnabled(cacheable=true)
    public static List<Employee__c> getAllEmployees() {
        return [
            SELECT Id, Employee_Name__c, Employee_ID__c,
                   Salary__c, Email__c, Department__c, Joining_Date__c
            FROM Employee__c
            ORDER BY CreatedDate DESC
        ];
    }

    // Check if Employee ID already exists (for uniqueness validation)
    @AuraEnabled(cacheable=false)
    public static Boolean isEmployeeIdUnique(Decimal empId) {
        List<Employee__c> existing = [
            SELECT Id FROM Employee__c
            WHERE Employee_ID__c = :empId
            LIMIT 1
        ];
        return existing.isEmpty();
    }

    // Save new employee record
    @AuraEnabled
    public static String saveEmployee(
        String  empName,
        Decimal empId,
        Decimal salary,
        String  email,
        String  department,
        Date    joiningDate
    ) {
        try {
            // Double-check uniqueness on server side
            List<Employee__c> existing = [
                SELECT Id FROM Employee__c
                WHERE Employee_ID__c = :empId
                LIMIT 1
            ];
            if (!existing.isEmpty()) {
                throw new AuraHandledException(
                    'Employee ID ' + empId + ' already exists. Please use a unique ID.'
                );
            }

            Employee__c emp = new Employee__c(
                Employee_Name__c = empName,
                Employee_ID__c   = empId,
                Salary__c        = salary,
                Email__c         = email,
                Department__c    = department,
                Joining_Date__c  = joiningDate
            );

            insert emp;
            return 'SUCCESS';

        } catch (AuraHandledException e) {
            throw e;
        } catch (Exception e) {
            throw new AuraHandledException('Error saving employee: ' + e.getMessage());
        }
    }

    // Delete employee record
    @AuraEnabled
    public static String deleteEmployee(String recordId) {
        try {
            Employee__c emp = [SELECT Id FROM Employee__c WHERE Id = :recordId LIMIT 1];
            delete emp;
            return 'SUCCESS';
        } catch (Exception e) {
            throw new AuraHandledException('Error deleting employee: ' + e.getMessage());
        }
    }

    // Get department picklist values dynamically
    @AuraEnabled(cacheable=true)
    public static List<String> getDepartments() {
        List<String> departments = new List<String>();
        Schema.DescribeFieldResult fieldResult =
            Employee__c.Department__c.getDescribe();
        List<Schema.PicklistEntry> entries = fieldResult.getPicklistValues();
        for (Schema.PicklistEntry entry : entries) {
            if (entry.isActive()) {
                departments.add(entry.getValue());
            }
        }
        return departments;
    }
}
```

---

## STEP 3 — Create Lightning Web Component

### Using VS Code + Salesforce CLI:

```bash
# Create LWC component
sf lightning generate component --name employeeManagement --type lwc

# This creates:
# force-app/main/default/lwc/employeeManagement/
#   ├── employeeManagement.html
#   ├── employeeManagement.js
#   ├── employeeManagement.css
#   └── employeeManagement.js-meta.xml
```

---

### File 1: `employeeManagement.html`

```html
<template>
    <lightning-card title="Employee Management System" icon-name="standard:employee">

        <!-- Toast Notification Area -->
        <template if:true={showToast}>
            <div class={toastClass} role="alert">
                <span class="toast-icon">{toastIcon}</span>
                <span class="toast-message">{toastMessage}</span>
                <button class="toast-close" onclick={closeToast}>✕</button>
            </div>
        </template>

        <div class="slds-p-around_medium">

            <!-- ===== ADD EMPLOYEE FORM ===== -->
            <lightning-card title="Add New Employee" icon-name="utility:add">
                <div class="slds-p-around_medium">
                    <div class="slds-grid slds-wrap slds-gutters">

                        <!-- Employee Name -->
                        <div class="slds-col slds-size_1-of-1 slds-medium-size_1-of-2">
                            <lightning-input
                                type="text"
                                label="Employee Name"
                                placeholder="Enter full name (min 3 characters)"
                                value={formData.empName}
                                onchange={handleInputChange}
                                data-field="empName"
                                required
                                message-when-value-missing="Employee Name is required."
                                class={getFieldClass('empName')}
                            ></lightning-input>
                            <template if:true={errors.empName}>
                                <p class="error-text">⚠ {errors.empName}</p>
                            </template>
                        </div>

                        <!-- Employee ID -->
                        <div class="slds-col slds-size_1-of-1 slds-medium-size_1-of-2">
                            <lightning-input
                                type="number"
                                label="Employee ID"
                                placeholder="Enter unique ID (e.g. 1001)"
                                value={formData.empId}
                                onchange={handleInputChange}
                                data-field="empId"
                                required
                                min="1"
                                class={getFieldClass('empId')}
                            ></lightning-input>
                            <template if:true={errors.empId}>
                                <p class="error-text">⚠ {errors.empId}</p>
                            </template>
                        </div>

                        <!-- Salary -->
                        <div class="slds-col slds-size_1-of-1 slds-medium-size_1-of-2">
                            <lightning-input
                                type="number"
                                label="Salary (₹)"
                                placeholder="10,000 – 5,00,000"
                                value={formData.salary}
                                onchange={handleInputChange}
                                data-field="salary"
                                required
                                formatter="currency"
                                class={getFieldClass('salary')}
                            ></lightning-input>
                            <template if:true={errors.salary}>
                                <p class="error-text">⚠ {errors.salary}</p>
                            </template>
                        </div>

                        <!-- Email -->
                        <div class="slds-col slds-size_1-of-1 slds-medium-size_1-of-2">
                            <lightning-input
                                type="email"
                                label="Email Address"
                                placeholder="employee@company.com"
                                value={formData.email}
                                onchange={handleInputChange}
                                data-field="email"
                                required
                                class={getFieldClass('email')}
                            ></lightning-input>
                            <template if:true={errors.email}>
                                <p class="error-text">⚠ {errors.email}</p>
                            </template>
                        </div>

                        <!-- Department -->
                        <div class="slds-col slds-size_1-of-1 slds-medium-size_1-of-2">
                            <lightning-combobox
                                label="Department"
                                placeholder="-- Select Department --"
                                value={formData.department}
                                options={departmentOptions}
                                onchange={handleInputChange}
                                data-field="department"
                                required
                                class={getFieldClass('department')}
                            ></lightning-combobox>
                            <template if:true={errors.department}>
                                <p class="error-text">⚠ {errors.department}</p>
                            </template>
                        </div>

                        <!-- Joining Date -->
                        <div class="slds-col slds-size_1-of-1 slds-medium-size_1-of-2">
                            <lightning-input
                                type="date"
                                label="Joining Date"
                                value={formData.joiningDate}
                                onchange={handleInputChange}
                                data-field="joiningDate"
                                required
                                max={todayDate}
                                class={getFieldClass('joiningDate')}
                            ></lightning-input>
                            <template if:true={errors.joiningDate}>
                                <p class="error-text">⚠ {errors.joiningDate}</p>
                            </template>
                        </div>

                    </div>

                    <!-- Action Buttons -->
                    <div class="slds-m-top_medium button-group">
                        <lightning-button
                            variant="brand"
                            label="Add Employee"
                            icon-name="utility:save"
                            onclick={handleSubmit}
                            disabled={isLoading}
                        ></lightning-button>
                        <lightning-button
                            variant="neutral"
                            label="Clear Form"
                            icon-name="utility:clear"
                            onclick={handleClear}
                            class="slds-m-left_small"
                        ></lightning-button>
                    </div>

                    <!-- Loading Spinner -->
                    <template if:true={isLoading}>
                        <div class="spinner-container">
                            <lightning-spinner
                                alternative-text="Saving..."
                                size="medium"
                                variant="brand"
                            ></lightning-spinner>
                            <p class="loading-text">Saving employee record...</p>
                        </div>
                    </template>

                </div>
            </lightning-card>

            <!-- ===== EMPLOYEE TABLE ===== -->
            <div class="slds-m-top_medium">
                <lightning-card title={tableTitle} icon-name="standard:employee">
                    <div class="slds-p-around_medium">

                        <template if:true={hasEmployees}>
                            <div class="table-container">
                                <table class="slds-table slds-table_cell-buffer slds-table_bordered slds-table_striped">
                                    <thead>
                                        <tr class="slds-line-height_reset">
                                            <th>Emp ID</th>
                                            <th>Name</th>
                                            <th>Email</th>
                                            <th>Department</th>
                                            <th>Salary (₹)</th>
                                            <th>Joining Date</th>
                                            <th>Action</th>
                                        </tr>
                                    </thead>
                                    <tbody>
                                        <template for:each={employees} for:item="emp">
                                            <tr key={emp.Id}>
                                                <td>{emp.Employee_ID__c}</td>
                                                <td><strong>{emp.Employee_Name__c}</strong></td>
                                                <td>{emp.Email__c}</td>
                                                <td>
                                                    <lightning-badge label={emp.Department__c}></lightning-badge>
                                                </td>
                                                <td>₹{emp.Salary__c}</td>
                                                <td>{emp.Joining_Date__c}</td>
                                                <td>
                                                    <lightning-button-icon
                                                        icon-name="utility:delete"
                                                        variant="destructive"
                                                        alternative-text="Delete"
                                                        data-id={emp.Id}
                                                        onclick={handleDelete}
                                                        title="Delete Employee"
                                                    ></lightning-button-icon>
                                                </td>
                                            </tr>
                                        </template>
                                    </tbody>
                                </table>
                            </div>
                        </template>

                        <template if:false={hasEmployees}>
                            <div class="empty-state">
                                <p>👥 No employees found. Add your first employee using the form above.</p>
                            </div>
                        </template>

                    </div>
                </lightning-card>
            </div>

        </div>
    </lightning-card>
</template>
```

---

### File 2: `employeeManagement.js`

```javascript
import { LightningElement, track, wire } from 'lwc';
import { ShowToastEvent } from 'lightning/platformShowToastEvent';
import { refreshApex } from '@salesforce/apex';

import getAllEmployees   from '@salesforce/apex/EmployeeController.getAllEmployees';
import isEmployeeIdUnique from '@salesforce/apex/EmployeeController.isEmployeeIdUnique';
import saveEmployee     from '@salesforce/apex/EmployeeController.saveEmployee';
import deleteEmployee   from '@salesforce/apex/EmployeeController.deleteEmployee';
import getDepartments   from '@salesforce/apex/EmployeeController.getDepartments';

export default class EmployeeManagement extends LightningElement {

    // ─── Reactive State ───────────────────────────────────
    @track formData = {
        empName    : '',
        empId      : '',
        salary     : '',
        email      : '',
        department : '',
        joiningDate: ''
    };

    @track errors = {
        empName    : '',
        empId      : '',
        salary     : '',
        email      : '',
        department : '',
        joiningDate: ''
    };

    @track employees        = [];
    @track departmentOptions = [];
    @track isLoading        = false;
    @track showToast        = false;
    @track toastMessage     = '';
    @track toastClass       = '';
    @track toastIcon        = '';

    wiredEmployeesResult;

    // ─── Today's Date (max for joining date) ──────────────
    get todayDate() {
        return new Date().toISOString().split('T')[0];
    }

    get tableTitle() {
        return `Employee Records (${this.employees.length})`;
    }

    get hasEmployees() {
        return this.employees && this.employees.length > 0;
    }

    // ─── Wire: Load Employees ──────────────────────────────
    @wire(getAllEmployees)
    wiredEmployees(result) {
        this.wiredEmployeesResult = result;
        if (result.data) {
            this.employees = result.data;
        } else if (result.error) {
            this.showToastNotification('Error', 'Failed to load employees.', 'error');
        }
    }

    // ─── Wire: Load Departments ───────────────────────────
    @wire(getDepartments)
    wiredDepartments({ data, error }) {
        if (data) {
            this.departmentOptions = data.map(dept => ({
                label: dept,
                value: dept
            }));
        } else if (error) {
            // Fallback hardcoded departments
            this.departmentOptions = [
                { label: 'HR',          value: 'HR' },
                { label: 'Engineering', value: 'Engineering' },
                { label: 'Finance',     value: 'Finance' },
                { label: 'Marketing',   value: 'Marketing' },
                { label: 'Sales',       value: 'Sales' },
                { label: 'Operations',  value: 'Operations' },
                { label: 'IT',          value: 'IT' },
                { label: 'Legal',       value: 'Legal' }
            ];
        }
    }

    // ─── Handle Input Changes ─────────────────────────────
    handleInputChange(event) {
        const field = event.target.dataset.field;
        this.formData[field] = event.target.value;
        // Clear error for this field on change
        this.errors[field] = '';
    }

    // ─── Validation ───────────────────────────────────────
    validateForm() {
        let isValid = true;
        const errors = {
            empName: '', empId: '', salary: '',
            email: '', department: '', joiningDate: ''
        };

        // 1. Employee Name
        const name = this.formData.empName.trim();
        if (!name) {
            errors.empName = 'Employee Name is required.';
            isValid = false;
        } else if (name.length < 3) {
            errors.empName = 'Employee Name must be at least 3 characters long.';
            isValid = false;
        } else if (!/^[a-zA-Z\s]+$/.test(name)) {
            errors.empName = 'Employee Name must contain only letters and spaces.';
            isValid = false;
        }

        // 2. Employee ID
        const empId = Number(this.formData.empId);
        if (!this.formData.empId) {
            errors.empId = 'Employee ID is required.';
            isValid = false;
        } else if (!Number.isInteger(empId) || empId <= 0) {
            errors.empId = 'Employee ID must be a positive whole number greater than 0.';
            isValid = false;
        }

        // 3. Salary
        const salary = Number(this.formData.salary);
        if (!this.formData.salary) {
            errors.salary = 'Salary is required.';
            isValid = false;
        } else if (salary <= 10000) {
            errors.salary = 'Salary must be greater than ₹10,000.';
            isValid = false;
        } else if (salary >= 500000) {
            errors.salary = 'Salary must be less than ₹5,00,000.';
            isValid = false;
        }

        // 4. Email
        const emailRegex = /^[a-zA-Z0-9._+-]+@[a-zA-Z0-9-]+\.[a-zA-Z]{2,}$/;
        if (!this.formData.email) {
            errors.email = 'Email address is required.';
            isValid = false;
        } else if (!emailRegex.test(this.formData.email)) {
            errors.email = 'Please enter a valid email address (e.g. user@company.com).';
            isValid = false;
        }

        // 5. Department
        if (!this.formData.department) {
            errors.department = 'Please select a department from the list.';
            isValid = false;
        }

        // 6. Joining Date
        if (!this.formData.joiningDate) {
            errors.joiningDate = 'Joining Date is required.';
            isValid = false;
        } else {
            const today    = new Date();
            today.setHours(0, 0, 0, 0);
            const joinDate = new Date(this.formData.joiningDate);
            if (joinDate > today) {
                errors.joiningDate = 'Joining Date cannot be a future date.';
                isValid = false;
            }
        }

        this.errors = errors;
        return isValid;
    }

    // ─── Submit Handler ───────────────────────────────────
    async handleSubmit() {
        // Step 1: Client-side validation
        if (!this.validateForm()) {
            this.showToastNotification(
                'Validation Failed',
                'Please fix the errors before submitting.',
                'error'
            );
            return;
        }

        this.isLoading = true;

        try {
            // Step 2: Check Employee ID uniqueness (server-side)
            const isUnique = await isEmployeeIdUnique({
                empId: Number(this.formData.empId)
            });

            if (!isUnique) {
                this.errors.empId =
                    `Employee ID ${this.formData.empId} already exists. Please use a unique ID.`;
                this.showToastNotification(
                    'Duplicate ID',
                    `Employee ID ${this.formData.empId} is already taken.`,
                    'error'
                );
                this.isLoading = false;
                return;
            }

            // Step 3: Save to Salesforce
            const result = await saveEmployee({
                empName    : this.formData.empName.trim(),
                empId      : Number(this.formData.empId),
                salary     : Number(this.formData.salary),
                email      : this.formData.email,
                department : this.formData.department,
                joiningDate: this.formData.joiningDate
            });

            if (result === 'SUCCESS') {
                this.showToastNotification(
                    'Employee Added!',
                    `${this.formData.empName} has been successfully added.`,
                    'success'
                );
                this.handleClear();
                // Refresh employee list
                await refreshApex(this.wiredEmployeesResult);
            }

        } catch (error) {
            const msg = error.body ? error.body.message : error.message;
            this.showToastNotification('Error', msg, 'error');
        } finally {
            this.isLoading = false;
        }
    }

    // ─── Delete Handler ───────────────────────────────────
    async handleDelete(event) {
        const recordId = event.currentTarget.dataset.id;
        if (!confirm('Are you sure you want to delete this employee?')) return;

        this.isLoading = true;
        try {
            await deleteEmployee({ recordId });
            this.showToastNotification('Deleted', 'Employee record deleted.', 'success');
            await refreshApex(this.wiredEmployeesResult);
        } catch (error) {
            const msg = error.body ? error.body.message : error.message;
            this.showToastNotification('Error', msg, 'error');
        } finally {
            this.isLoading = false;
        }
    }

    // ─── Clear Form ───────────────────────────────────────
    handleClear() {
        this.formData = {
            empName: '', empId: '', salary: '',
            email: '', department: '', joiningDate: ''
        };
        this.errors = {
            empName: '', empId: '', salary: '',
            email: '', department: '', joiningDate: ''
        };
    }

    // ─── Field CSS Class (error highlight) ────────────────
    getFieldClass(field) {
        return this.errors[field] ? 'input-error' : '';
    }

    // ─── Toast Notification ───────────────────────────────
    showToastNotification(title, message, variant) {
        // Native Salesforce toast
        this.dispatchEvent(new ShowToastEvent({ title, message, variant }));
        // Custom in-component toast (fallback)
        this.toastMessage = message;
        this.showToast    = true;
        this.toastIcon    = variant === 'success' ? '✅' : '❌';
        this.toastClass   = variant === 'success'
            ? 'custom-toast toast-success'
            : 'custom-toast toast-error';
        // Auto-hide after 4 seconds
        setTimeout(() => { this.showToast = false; }, 4000);
    }

    closeToast() {
        this.showToast = false;
    }
}
```

---

### File 3: `employeeManagement.css`

```css
/* ── Container ── */
.table-container {
    overflow-x: auto;
}

/* ── Error Text ── */
.error-text {
    color: #c23934;
    font-size: 12px;
    margin: 4px 0 0 0;
    font-weight: 500;
}

/* ── Input Error Highlight ── */
.input-error lightning-input,
.input-error lightning-combobox {
    --sds-c-input-color-border: #c23934;
    --sds-c-input-shadow-focus: 0 0 0 3px rgba(194,57,52,0.25);
}

/* ── Button Group ── */
.button-group {
    display: flex;
    align-items: center;
    gap: 10px;
    margin-top: 20px;
}

/* ── Spinner ── */
.spinner-container {
    display: flex;
    flex-direction: column;
    align-items: center;
    margin-top: 20px;
}
.loading-text {
    margin-top: 40px;
    color: #0070d2;
    font-size: 14px;
}

/* ── Empty State ── */
.empty-state {
    text-align: center;
    padding: 40px;
    color: #706e6b;
    font-size: 16px;
}

/* ── Table ── */
table th {
    background: #f3f3f3;
    font-weight: 700;
    font-size: 13px;
    padding: 10px 12px;
    white-space: nowrap;
    color: #3e3e3c;
}
table td {
    padding: 10px 12px;
    font-size: 13px;
    vertical-align: middle;
}

/* ── Custom Toast ── */
.custom-toast {
    display: flex;
    align-items: center;
    padding: 12px 16px;
    border-radius: 4px;
    margin-bottom: 16px;
    font-size: 14px;
    font-weight: 500;
    gap: 10px;
    position: relative;
}
.toast-success {
    background: #e8f5e9;
    border-left: 4px solid #2e7d32;
    color: #2e7d32;
}
.toast-error {
    background: #fdecea;
    border-left: 4px solid #c23934;
    color: #c23934;
}
.toast-close {
    background: none;
    border: none;
    font-size: 16px;
    cursor: pointer;
    position: absolute;
    right: 10px;
    color: inherit;
}
.toast-message {
    flex: 1;
}
```

---

### File 4: `employeeManagement.js-meta.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<LightningComponentBundle xmlns="http://soap.sforce.com/2006/04/metadata">
    <apiVersion>59.0</apiVersion>
    <isExposed>true</isExposed>
    <targets>
        <target>lightning__AppPage</target>
        <target>lightning__RecordPage</target>
        <target>lightning__HomePage</target>
        <target>lightning__Tab</target>
    </targets>
    <targetConfigs>
        <targetConfig targets="lightning__AppPage">
            <property name="title" type="String" label="Component Title"
                      default="Employee Management"/>
        </targetConfig>
    </targetConfigs>
</LightningComponentBundle>
```

---

## STEP 4 — Deploy to Salesforce

```bash
# Authenticate to your org
sf org login web --alias myOrg

# Deploy all files
sf project deploy start --source-dir force-app

# Or deploy only LWC + Apex
sf project deploy start \
  --source-dir force-app/main/default/lwc/employeeManagement \
  --source-dir force-app/main/default/classes/EmployeeController.cls
```

---

## STEP 5 — Add Component to a Page

1. Go to **Setup → Lightning App Builder**
2. Click **"New"** → **App Page** → name it "Employee Manager"
3. Choose layout: **One Region**
4. From left panel, search **"employeeManagement"**
5. Drag it onto the canvas
6. Click **"Save"** → **"Activate"** → **"Activate for all users"**
7. Click **"Finish"**

---

## Project Folder Structure

```
force-app/main/default/
├── classes/
│   ├── EmployeeController.cls
│   └── EmployeeController.cls-meta.xml
├── lwc/
│   └── employeeManagement/
│       ├── employeeManagement.html
│       ├── employeeManagement.js
│       ├── employeeManagement.css
│       └── employeeManagement.js-meta.xml
└── objects/
    └── Employee__c/
        ├── Employee__c.object-meta.xml
        └── fields/
            ├── Employee_Name__c.field-meta.xml
            ├── Employee_ID__c.field-meta.xml
            ├── Salary__c.field-meta.xml
            ├── Email__c.field-meta.xml
            ├── Department__c.field-meta.xml
            └── Joining_Date__c.field-meta.xml
```

---

## Validation Rules Summary

| Field | Validation Rule | Error Message |
|-------|----------------|---------------|
| Employee Name | Cannot be empty | "Employee Name is required." |
| Employee Name | Min 3 characters | "Employee Name must be at least 3 characters long." |
| Employee Name | Letters only | "Employee Name must contain only letters and spaces." |
| Employee ID | Cannot be empty | "Employee ID is required." |
| Employee ID | Must be > 0 | "Employee ID must be a positive whole number greater than 0." |
| Employee ID | Must be unique | "Employee ID already exists. Please use a unique ID." |
| Salary | Cannot be empty | "Salary is required." |
| Salary | Must be > 10,000 | "Salary must be greater than ₹10,000." |
| Salary | Must be < 500,000 | "Salary must be less than ₹5,00,000." |
| Email | Cannot be empty | "Email address is required." |
| Email | Valid format | "Please enter a valid email address (e.g. user@company.com)." |
| Department | Must be selected | "Please select a department from the list." |
| Joining Date | Cannot be empty | "Joining Date is required." |
| Joining Date | Cannot be future | "Joining Date cannot be a future date." |

---

## Test Cases

| # | Scenario | Input | Expected Result |
|---|----------|-------|-----------------|
| 1 | Valid data, no issues | All valid values | ✅ Employee saved, appears in table |
| 2 | Empty name | Name: "" | ❌ "Employee Name is required" |
| 3 | Short name | Name: "Ab" | ❌ "At least 3 characters" |
| 4 | Negative Employee ID | ID: -5 | ❌ "Must be greater than 0" |
| 5 | Duplicate Employee ID | ID: already exists | ❌ "ID already exists" |
| 6 | Low salary | Salary: 5000 | ❌ "Must be greater than ₹10,000" |
| 7 | High salary | Salary: 600000 | ❌ "Must be less than ₹5,00,000" |
| 8 | Invalid email | Email: "abc@" | ❌ "Valid email format required" |
| 9 | No department | Department: none | ❌ "Please select a department" |
| 10 | Future date | Date: tomorrow | ❌ "Cannot be a future date" |
| 11 | Delete employee | Click delete icon | ✅ Record removed from table |

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| Component not visible in App Builder | Check `isExposed: true` in `.js-meta.xml` |
| Apex method not found | Ensure `@AuraEnabled` annotation is present |
| Wire not loading data | Add `cacheable=true` to `@AuraEnabled` on getter methods |
| Employee ID not checking uniqueness | Use `cacheable=false` on `isEmployeeIdUnique` method |
| Deploy failing | Run `sf org login web` to re-authenticate |
| Department picklist empty | Verify picklist values are active in Object Manager |
| Table not refreshing after add | Ensure `refreshApex(this.wiredEmployeesResult)` is called |
| Toast not showing | Check `ShowToastEvent` import from `lightning/platformShowToastEvent` |

---

## Quick Reference — Key LWC Concepts Used

```javascript
// Wire service — auto-fetch data
@wire(getAllEmployees) wiredEmployees(result) { ... }

// Refresh wired data after DML
await refreshApex(this.wiredEmployeesResult);

// Call Apex imperatively
const result = await saveEmployee({ empName, empId, ... });

// Show toast notification
this.dispatchEvent(new ShowToastEvent({ title, message, variant }));

// Track reactive state
@track formData = { empName: '', empId: '' ... };
```
# Student Management System - Salesforce Apex & Visualforce

## Apex Controller: StudentController.cls

```java
public class StudentController {

    public Student__c stu {get; set;}
    public List<Student__c> studentList {get; set;}

    public StudentController(){
        stu = new Student__c();

        studentList = [
            SELECT Id, Name, Roll_No__c, Class__c, Mobile_No__c
            FROM Student__c
            ORDER BY Roll_No__c
        ];
    }

    // CREATE
    public PageReference saveStudent(){

        insert stu;

        return Page.StudentListPage;
    }

    // EDIT PAGE
    public PageReference editStudent(){

        Id sid = ApexPages.currentPage()
        .getParameters().get('sid');

        stu = [
            SELECT Id, Name, Roll_No__c,
            Class__c, Mobile_No__c
            FROM Student__c
            WHERE Id=:sid
        ];

        return Page.UpdateStudentPage;
    }

    // UPDATE
    public PageReference updateStudent(){

        update stu;

        return Page.StudentListPage;
    }

    // DELETE
    public PageReference deleteStudent(){

        Id sid=ApexPages.currentPage()
        .getParameters().get('sid');

        Student__c s=
        [SELECT Id FROM Student__c WHERE Id=:sid];

        delete s;

        return Page.StudentListPage;
    }
}
<apex:page controller="StudentController">

<apex:form>

<apex:pageBlock title="Student Management System">

<apex:pageBlockSection columns="1">

<apex:inputField value="{!stu.Name}"/>

<apex:inputField value="{!stu.Roll_No__c}"/>

<apex:inputField value="{!stu.Class__c}"/>

<apex:inputField value="{!stu.Mobile_No__c}"/>

</apex:pageBlockSection>

<apex:commandButton
value="Save"
action="{!saveStudent}"/>

<apex:commandButton
value="View Records"
action="{!StudentListPage}"/>

</apex:pageBlock>

</apex:form>

</apex:page>
