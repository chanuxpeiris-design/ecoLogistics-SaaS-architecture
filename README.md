# 🌍 EcoLogistics: Multi-Tier Fleet Management and CO2 Tracking Engine
**A Comprehensive Build-in-Public Guide to Architecting a Green-Tech SaaS**

*Note: This repository serves as a live, "build-in-public" white paper and technical guide. It outlines the step-by-step architectural decisions, database schemas, and core algorithms required to build a scalable, multi-role logistics platform from scratch.*

## 📌 Part 1: Executive Summary and Problem Statement

### The Market Gap
Modern corporate fleets face a dual challenge: managing operational logistics (drivers, routes, deliveries) while simultaneously tracking and reporting their carbon footprint (CO2 emissions) to meet increasingly strict environmental regulations. Existing solutions are often fragmented, requiring separate software for fleet management and environmental compliance.

### The EcoLogistics Solution
EcoLogistics is a unified SaaS platform designed to bridge this gap. It provides a real-time tracking interface for drivers while aggregating operational data into a dynamic CO2 emission calculation engine for managers. 



## 🏗️ Part 1: System Architecture and Tech Stack

To ensure high performance, scalability, and cross-platform compatibility, the system is designed utilizing a modern BaaS (Backend-as-a-Service) architecture.

### Core Technologies
* **Frontend Client:** Flutter / Dart (Cross-platform compatibility for both Web-based Manager Dashboards and Mobile-based Driver Apps).
* **Backend and API:** Supabase (Handling Auth, Edge Functions, and Real-time subscriptions).
* **Database:** PostgreSQL (Strictly relational data modeling to handle complex entity relationships).

### Multi-Tier Role System (RBAC)
The platform is built on a strict Role-Based Access Control framework, serving two distinct client interfaces from a single unified backend:
1.  **The Manager Portal (Web):** A data-heavy dashboard for monitoring fleet-wide metrics, managing driver accounts via secure company codes, and viewing aggregated CO2 reports.
2.  **The Driver App (Mobile):** A streamlined interface for logging trip distances, vehicle types, and delivery statuses.

## 🚀 Development Roadmap
This repository is structured as a chronological guide to building the platform:
- [x] Phase 1: System Architecture and Tech Stack Evaluation
- [ ] Phase 2: Database Schema and Relational Modeling
- [ ] Phase 3: Auth, RBAC, and Row Level Security (RLS)
- [ ] Phase 4: The CO2 Calculation Algorithm
- [ ] Phase 5: Client-Side Implementation

## 🗄️ Part 2: Relational Database Design and Schema Architecture

### The Data Integrity Strategy
A logistics platform dealing with environmental compliance cannot rely on unstructured data. If a driver is deleted, their historical CO2 emissions must remain intact for company reporting. If a company is deleted, all associated fleet data must be securely wiped. 

To enforce this strict data integrity, EcoLogistics utilizes **PostgreSQL** to build a highly relational structure with cascading rules and foreign key constraints.

### The Core Entity Relationship
The database is structured around four primary entities:
1.  **Companies:** The top-level tenant. 
2.  **Profiles:** The users (Managers and Drivers), linked to Supabase Auth.
3.  **Vehicles:** The physical fleet assets.
4.  **Trips:** The transactional logs where distance and CO2 are recorded.

### 📝 PostgreSQL Schema Implementation

Below is the foundational SQL architecture used to build the platform's relational logic.

#### 1. Defining Roles and the Tenant (Company)
We start by defining the strict user roles and the central `companies` table. Notice the `auth_code` column—this is the unique identifier managers give to drivers so they join the correct fleet upon registration.

```sql
-- Create a strict enum for Role-Based Access Control
CREATE TYPE user_role AS ENUM ('manager', 'driver');

-- Core tenant table
CREATE TABLE companies (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name VARCHAR(255) NOT NULL,
    auth_code VARCHAR(10) UNIQUE NOT NULL, 
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

#### 2. User Profiles and Authentication Linking
Supabase handles the actual passwords in a secure, hidden auth.users schema. We create a public profiles table that references that secure schema, assigning the user to a company and a specific role.

```sql
CREATE TABLE profiles (
    id UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
    company_id UUID REFERENCES companies(id) ON DELETE RESTRICT,
    full_name VARCHAR(255) NOT NULL,
    role user_role NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```
Architecture Note: ON DELETE RESTRICT is used for company_id to prevent the accidental deletion of a company if active users are still attached to it.

#### 3. Fleet and Emission Factors
Every vehicle in the fleet is assigned an emission_factor (representing kg of CO2 emitted per kilometer). This allows the database to dynamically calculate emissions later based on the specific truck driven, rather than using a static average.

```sql
CREATE TABLE vehicles (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    company_id UUID REFERENCES companies(id) ON DELETE CASCADE,
    license_plate VARCHAR(50) NOT NULL,
    emission_factor DECIMAL(5,2) NOT NULL, 
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

#### 4. The Transactional Layer: Trip Logging
This is where the actual daily data is written by the Driver App. It links the driver, the specific vehicle, and the distance traveled.

```sql
CREATE TABLE trips (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    driver_id UUID REFERENCES profiles(id) ON DELETE RESTRICT,
    vehicle_id UUID REFERENCES vehicles(id) ON DELETE RESTRICT,
    distance_km DECIMAL(10,2) NOT NULL,
    trip_date DATE NOT NULL DEFAULT CURRENT_DATE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

## 🔐 Part 3: Authentication and Role-Based Access Control (RBAC)

### The Security Philosophy
A multi-tier SaaS handling sensitive corporate logistics cannot rely on frontend routing for security. If a user modifies their local application state or attempts to bypass the UI, they must still be blocked by the server. EcoLogistics enforces strict Role-Based Access Control (RBAC) directly at the database and authentication layer utilizing Supabase Auth.

### The Authentication Flow and Company Linking
The system must seamlessly handle two distinct user onboarding flows without manual administrative intervention:
1.  **The Fleet Manager:** Creates a new company account and is issued a unique, secure `auth_code`.
2.  **The Driver:** Downloads the mobile application, creates an account, and inputs the `auth_code` provided by their manager. This automatically binds their profile to the correct company tenant and restricts their permissions to the "driver" role.

### 📝 PostgreSQL Trigger: Automated Profile Provisioning

Instead of relying on the Flutter client application to execute multiple sequential database inserts (which can fail and create orphan records if the user loses internet connection during signup), the system uses a PostgreSQL Database Trigger. 

When a new user registers via Supabase Auth, the server automatically executes a function to create their public profile and assign their role based on the presence of an `auth_code`.

```sql
-- 1. Define the function that runs automatically on new user signup
CREATE OR REPLACE FUNCTION public.handle_new_user() 
RETURNS TRIGGER AS $$
DECLARE
    passed_auth_code VARCHAR;
    target_company_id UUID;
    assigned_role user_role;
BEGIN
    -- Extract the auth code passed from the frontend metadata during signup
    passed_auth_code := new.raw_user_meta_data->>'auth_code';

    IF passed_auth_code IS NOT NULL THEN
        -- Driver Flow: Look up the company by the provided auth code
        SELECT id INTO target_company_id FROM public.companies WHERE auth_code = passed_auth_code LIMIT 1;
        
        IF target_company_id IS NULL THEN
            RAISE EXCEPTION 'Invalid company authorization code.';
        END IF;
        
        assigned_role := 'driver'::user_role;
    ELSE
        -- Manager Flow: If no code is passed, they are a manager creating a new tenant
        assigned_role := 'manager'::user_role;
        target_company_id := NULL; -- Will be updated once they complete company setup
    END IF;

    -- Automatically create the public profile with the securely determined role
    INSERT INTO public.profiles (id, company_id, full_name, role)
    VALUES (new.id, target_company_id, new.raw_user_meta_data->>'full_name', assigned_role);

    RETURN new;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- 2. Bind the function to the Supabase Auth schema
CREATE TRIGGER on_auth_user_created
    AFTER INSERT ON auth.users
    FOR EACH ROW EXECUTE PROCEDURE public.handle_new_user();
```

By using a SECURITY DEFINER trigger, the backend handles the heavy lifting safely. The frontend never has to be trusted with manually writing to the profiles table or assigning its own role. It simply passes the user's name and the optional company code during the initial email and password signup, and the PostgreSQL server orchestrates the rest securely.
With users now authenticated and locked into their respective roles and companies, the platform must enforce strict data isolation.


## 🛡️ Part 4: Row Level Security (RLS) Implementation Guide

### The Multi-Tenant Data Isolation Problem
In a SaaS architecture where multiple companies share the same database tables, absolute data isolation is critical. A manager from "Company A" must never be able to query the vehicles, drivers, or CO2 emissions of "Company B." Furthermore, a driver should only be able to log trips for themselves, not on behalf of other drivers.

Instead of writing complex authorization logic in the Flutter frontend or relying on edge functions to filter data, EcoLogistics enforces isolation at the lowest possible level: the database rows. 

### Enabling RLS and Defining Policies
By enabling Row Level Security (RLS) in PostgreSQL, every single query—whether from a mobile app, a web dashboard, or an API call—must pass a strict policy check before the database returns any data.

#### 1. Securing the Profiles Table
First, we enable RLS on the `profiles` table. The policy ensures that a user can only read profiles that belong to their specific `company_id`.

```sql
-- Enable RLS on the table
ALTER TABLE public.profiles ENABLE ROW LEVEL SECURITY;

-- Policy: Users can only see profiles within their own company
CREATE POLICY "Users can view company profiles" 
ON public.profiles 
FOR SELECT 
USING (
    company_id = (
        SELECT company_id FROM public.profiles WHERE id = auth.uid()
    )
);

-- Policy: Users can only update their own specific profile
CREATE POLICY "Users can update own profile" 
ON public.profiles 
FOR UPDATE 
USING (id = auth.uid());## 🛡️ Part 4: Row Level Security (RLS) Implementation Guide
```

#### 2. Locking Down the Vehicles
Managers need the ability to add, update, and delete vehicles for their fleet. Drivers only need to view the vehicles to select them when logging a trip.

```sql
ALTER TABLE public.vehicles ENABLE ROW LEVEL SECURITY;

-- Policy: Everyone in the company can view the company vehicles
CREATE POLICY "Company members can view vehicles" 
ON public.vehicles 
FOR SELECT 
USING (
    company_id = (
        SELECT company_id FROM public.profiles WHERE id = auth.uid()
    )
);

-- Policy: Only managers can insert or modify vehicles
CREATE POLICY "Managers can manage vehicles" 
ON public.vehicles 
FOR ALL 
USING (
    'manager' = (
        SELECT role FROM public.profiles WHERE id = auth.uid()
    )
    AND
    company_id = (
        SELECT company_id FROM public.profiles WHERE id = auth.uid()
    )
);
```

#### 3. Securing the Trip Logs (The Transactional Data)
This is the most critical security layer. Drivers must be able to insert trips, but they should only be able to insert trips tied to their own driver_id. Managers need to read all trips for the company to calculate CO2.

```sql
ALTER TABLE public.trips ENABLE ROW LEVEL SECURITY;

-- Policy: Drivers can insert their own trips
CREATE POLICY "Drivers can insert own trips" 
ON public.trips 
FOR INSERT 
WITH CHECK (
    driver_id = auth.uid()
);

-- Policy: Managers can view all trips for their company's drivers
CREATE POLICY "Managers can view company trips" 
ON public.trips 
FOR SELECT 
USING (
    driver_id IN (
        SELECT id FROM public.profiles WHERE company_id = (
            SELECT company_id FROM public.profiles WHERE id = auth.uid()
        )
    )
);
```
The Architectural Result

With these policies active, the Flutter frontend becomes completely "dumb" regarding security. If a malicious user intercepts the API and tries to run SELECT * FROM vehicles, the PostgreSQL server will automatically restrict the response to only the vehicles matching their authenticated company_id.


## 🌍 Part 5: The CO2 Emission Calculation Engine (Core Logic)

### The Inefficiency of Client-Side Processing
The core feature of EcoLogistics is aggregating a company's total carbon footprint. A common mistake in SaaS development is fetching raw data (thousands of individual trip logs) to the frontend application and running the math locally. This consumes massive amounts of bandwidth, slows down the dashboard, and drains client resources.

Instead, the CO2 calculation engine is built directly into the PostgreSQL database using **Dynamic Views**. The server crunches the numbers, and the Flutter frontend simply fetches the final, aggregated results.

### The Calculation Formula
The math for Scope 1 emissions (direct fleet emissions) is straightforward: 
`Distance Traveled (km) * Vehicle Emission Factor (kg CO2/km) = Total CO2 (kg)`

### 📝 PostgreSQL Implementation: Dynamic Emission Views

We create a virtual table (a View) that automatically joins the `trips` data with the specific `vehicles` data to calculate the exact carbon footprint of every single trip the moment it is queried.

#### 1. The Individual Trip Emissions View
This view calculates the CO2 for each specific delivery or journey.

```sql
CREATE OR REPLACE VIEW trip_emissions AS
SELECT 
    t.id AS trip_id,
    t.trip_date,
    t.distance_km,
    v.license_plate,
    v.emission_factor,
    p.full_name AS driver_name,
    p.company_id,
    -- The Core Calculation Engine
    (t.distance_km * v.emission_factor) AS total_co2_kg
FROM 
    public.trips t
JOIN 
    public.vehicles v ON t.vehicle_id = v.id
JOIN 
    public.profiles p ON t.driver_id = p.id;
```

#### 2. The Fleet-Wide Aggregation View
Managers do not just want to see individual trips; they need to see the total carbon footprint of their entire fleet over time. We create a second view that aggregates the data from the first view, grouping it by company.

```sql
CREATE OR REPLACE VIEW company_emission_summary AS
SELECT 
    company_id,
    DATE_TRUNC('month', trip_date) AS emission_month,
    SUM(distance_km) AS total_fleet_distance_km,
    SUM(total_co2_kg) AS total_fleet_co2_kg
FROM 
    trip_emissions
GROUP BY 
    company_id, 
    DATE_TRUNC('month', trip_date)
ORDER BY 
    emission_month DESC;
```

Applying Security to Views

Because PostgreSQL Views bypass Row Level Security by default if not configured correctly, we must ensure these views are queried with the authenticated user's permissions. By structuring the queries through Supabase's authenticated API, the RLS policies built in Part 4 cascade down into these views. A manager querying the company_emission_summary will only receive the aggregated math for their specific company_id.

The database is now fully optimized to handle thousands of driver logs and instantly return calculated carbon reports.
