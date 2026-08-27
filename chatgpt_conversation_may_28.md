# Biker Shades Rebuild Notes (May 28, 2026)
**Date:** May 28, 2026

# Objective

Rebuild Biker Shades and eventually rebuild the owner's other two sunglasses brands while avoiding duplicated backend infrastructure and preparing for future AI/MCP integrations.

# Initial Discussion

The original idea was:

Brand 1 Frontend
Brand 2 Frontend
Brand 3 Frontend

with each frontend potentially containing its own backend and MCP server.

After discussion, this was determined to be inefficient because it would create duplicated business logic, duplicated APIs, duplicated inventory management, and duplicated MCP functionality.

# Architecture Exploration

## Option A: Single Frontend Application

One Next.js App
├── Biker Shades
├── Brand Two
└── Brand Three

All domains would point to the same application.

The application would inspect the incoming hostname and render the correct brand.

## Option B: Three Separate Frontends

Biker Shades Frontend
Brand Two Frontend
Brand Three Frontend

Each frontend would be a separate Next.js application with its own deployment.

# Final Decision

3 Frontends
1 Backend
1 MCP Server
1 Database

Architecture:

bikershades-site
brand-two-site
brand-three-site

        ↓

admin-api-mcp

        ↓

Supabase

Reasoning:
- Each brand will likely have a unique aesthetic.
- Each brand may eventually evolve independently.
- Managing multiple brands inside a single frontend would require increasing amounts of brand-specific conditional logic.
- Separate frontends are easier to reason about and maintain.
- Backend logic remains centralized.

# Backend Responsibilities

Products
Inventory
Categories
Images
Orders
Amazon Integration
MCP Tools
Admin Dashboard

Frontend → Backend API → Supabase

# MCP Server Design

The MCP server will live inside the backend deployment.

admin-api-mcp
├── API Routes
├── Admin Dashboard
└── MCP Routes

MCP should reuse backend business logic rather than directly accessing the database.

# Multi-Brand Database Design

The database should be designed for multiple brands from day one.

Core concept:

brand_id

Examples:

brands
products
categories
orders
inventory

Products should belong to brands.

Example schema:

products
---------
id
brand_id
title
description
price

# Frontend Hosting Discussion

## GitHub Pages

Can host:
- Static HTML
- CSS
- JavaScript

Cannot host:
- API Routes
- SSR
- Server Actions
- Node.js Code
- MCP Servers

## Client-Side Fetching

User
 ↓
Static Frontend
 ↓
fetch()
 ↓
Backend API
 ↓
Supabase

Example:

fetch("https://api.domain.com/products")

# SSR vs Client-Side Fetching

## Client-Side Fetching

Browser
 ↓
Loads page
 ↓
Fetches products
 ↓
Displays products

Pros:
- Works on GitHub Pages
- Simple architecture
- Cheap hosting

Cons:
- Product data loads after page render
- Potential SEO limitations

## Server-Side Rendering (SSR)

User
 ↓
Server
 ↓
Fetches products
 ↓
Generates HTML
 ↓
Returns page

Pros:
- Better SEO
- Better social sharing previews
- Immediate product visibility

Cons:
- Requires a server
- Cannot be hosted on GitHub Pages

# SEO Lessons Learned

Client-side rendering often sends Google minimal initial HTML before product content exists.

SSR sends product content immediately.

Key takeaway:

SSR is generally better for ecommerce SEO.

However, early traffic may come primarily from:
- Word of mouth
- Social media
- Amazon
- Existing customers

# Current Build Order

## Phase 1

Build:
- Biker Shades Frontend
- Shared Backend
- Shared MCP Server
- Supabase Database

while ensuring the backend and database are multi-brand aware.

## Phase 2

Deploy:
- bikershades.com
- api.domain.com

## Phase 3

Build Brand Two frontend.

Reuse:
- Backend
- Database
- MCP
- Admin Dashboard

## Phase 4

Build Brand Three frontend.

Reuse:
- Backend
- Database
- MCP
- Admin Dashboard

# Final Architecture

apps/
├── bikershades-site
├── brand-two-site
├── brand-three-site
└── admin-api-mcp

packages/
├── db
├── commerce
├── amazon
├── types
└── ui

Infrastructure
├── Supabase
├── Amazon SP-API
└── MCP Server

# Key Lessons Learned

1. Separate frontends do not require separate backends.
2. Backend logic should exist only once.
3. MCP should reuse backend services rather than duplicating logic.
4. Design the database for multiple brands from the beginning.
5. GitHub Pages can host static frontends but cannot host APIs or MCP servers.
6. Client-side fetching works well with a separated backend architecture.
7. SSR improves SEO but requires a server runtime.
8. The easiest path is to build Biker Shades first and validate the architecture before building the other two brands.
