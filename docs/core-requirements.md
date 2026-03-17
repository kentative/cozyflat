# Core Requirements

## AI Agents

### AI Maintenance Agent
- Multimedia triage (photo/video intake for maintenance requests)
- Autonomous dispatch via TaskRabbit and Thumbtack integrations
- Smart lock integration for vendor access management

### AI Financial Agent
- Smart ledgering for Schedule E tax reporting
- Predictive collections (rent delinquency forecasting and intervention)
- Automated payouts to property owners and vendors

### Compliance & Legal Agent
- Dynamic lease generation (jurisdiction-aware, auto-populated)
- Automated digital document serving (notices, renewals, disclosures)

### Communication Hub
- AI-moderated peer-to-peer chat between landlords and tenants
- Emergency escalation logic for urgent maintenance or safety issues

---

## Tech Stack

### Frontend
- **Web:** Next.js
- **Mobile:** React Native / Expo

### AI Layer
- **API:** FastAPI (Python)
- **Agent Orchestration:** LangGraph
- **Models:** GPT-4o, Claude 3.5

### Backend
- **Database:** Supabase (PostgreSQL)
- **Vector DB:** Pinecone (local laws and property manuals)

### Infrastructure
- **Payments:** Stripe Connect (peer-to-peer payments)
- **Background Checks:** Checkr
- **SMS / Voice:** Twilio
