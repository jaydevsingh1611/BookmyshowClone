QuickShow — Movie Ticket Booking Platform
A full-stack movie ticket booking platform built with React, Node.js, Express, MongoDB, Clerk, Stripe, TMDB, and Inngest.
The application supports movie discovery, show scheduling, seat selection, online payments, booking management, favorites, automated emails, and an admin dashboard.
Project note: The repository is named BookmyshowClone, while the application/backend code uses names such as QuickShow and netShow. The README uses QuickShow as the product name.
Features
Customer experience
- Browse currently available movies and movie details
- View ratings, genres, runtime, cast, overview, and release information
- View upcoming show dates and times
- Select seats for a specific show
- Check currently occupied seats
- Create a booking and pay through Stripe Checkout
- View booking history
- Regenerate payment links for eligible unpaid bookings
- Add/remove movies from favorites
- Receive booking confirmation emails
- Receive scheduled show reminders
Admin experience
- Admin dashboard with:
  - Total bookings
  - Total revenue
  - Active shows
  - Total users
- Fetch currently playing movies from TMDB
- Select a movie and schedule multiple show times
- Configure ticket prices
- View scheduled shows
- View bookings
- Delete shows associated with a movie
Backend capabilities
- REST API built with Express
- MongoDB persistence using Mongoose
- Clerk-based authentication
- Admin role validation through Clerk private metadata
- Stripe Checkout integration
- Stripe webhook handling for successful payments
- TMDB API integration
- Inngest event-driven/background workflows
- Nodemailer-based transactional email delivery
- Automatic release of unpaid seats after a timeout
- User synchronization between Clerk and MongoDB
Architecture
                         ┌─────────────────────┐
                         │       TMDB API      │
                         │ Movie metadata/data  │
                         └──────────┬──────────┘
                                    │
                                    ▼
┌──────────────────┐       ┌─────────────────────┐
│                  │       │                     │
│   React + Vite   │──────▶│  Express REST API   │
│                  │ HTTP  │                     │
└────────┬─────────┘       └───────┬─────────────┘
         │                         │
         │ Clerk                  │ Mongoose
         ▼                         ▼
┌──────────────────┐       ┌─────────────────────┐
│  Clerk Auth      │       │      MongoDB        │
│ Users / Roles    │       │ Movies / Shows /    │
└──────────────────┘       │ Bookings / Users    │
                           └──────────┬──────────┘
                                      │
                         ┌────────────┴────────────┐
                         │                         │
                         ▼                         ▼
                 ┌───────────────┐        ┌────────────────┐
                 │     Stripe    │        │    Inngest     │
                 │   Checkout    │        │ Background     │
                 │ + Webhooks    │        │ workflows/jobs │
                 └───────────────┘        └───────┬────────┘
                                                  │
                                                  ▼
                                           ┌─────────────┐
                                           │   SMTP      │
                                           │ Email/SMS*  │
                                           └─────────────┘
* Email delivery is implemented through SMTP using Nodemailer.
Tech Stack
Frontend
- React 19
- Vite
- React Router
- Tailwind CSS
- Clerk React
- Axios
- React Hot Toast
- Lucide React
- React Player
Backend
- Node.js
- Express 5
- Mongoose
- Clerk Express
- Stripe
- Inngest
- Axios
- Nodemailer
- CORS
Database & external services
- MongoDB
- TMDB API
- Stripe Checkout
- Clerk
- Inngest
- SMTP / Brevo
Deployment
- Vercel configuration is included for both client and server.
Core Data Model
The backend currently uses four main MongoDB models.
Movie
Stores movie metadata retrieved from TMDB.   
Movie
├── title
├── overview
├── poster_path
├── backdrop_path
├── release_date
├── original_language
├── tagline
├── genres
├── casts
├── vote_average
└── runtime         
Show
Represents a scheduled screening.  
Show
├── movie
├── showDateTime
├── showPrice
└── occupiedSeats                        
occupiedSeats stores seat-to-user mappings for the show.
Booking
Represents a customer's booking and payment state.
Booking
├── user
├── show
├── amount
├── bookedSeats
├── isPaid
├── paymentLink
├── createdAt
└── updatedAt
User
Stores synchronized user information from Clerk.
User
├── _id
├── name
├── email
└── image
Booking Flow
The booking flow is one of the main backend workflows in the project.
1. User selects a show
        ↓
2. Frontend requests occupied seats
        ↓
3. User selects available seats
        ↓
4. Backend validates seat availability
        ↓
5. Booking record is created
        ↓
6. Selected seats are marked as occupied
        ↓
7. Stripe Checkout session is created
        ↓
8. Payment URL is returned to frontend
        ↓
9. Inngest schedules payment-status check
        ↓
10. Stripe sends successful-payment webhook
        ↓
11. Booking is marked as paid
        ↓
12. Inngest sends confirmation email
Unpaid booking cleanup
If payment is not completed within the configured period:
Booking created
      ↓
Wait 10 minutes
      ↓
Check payment status
      ↓
Not paid?
      ↓
Release occupied seats
      ↓
Delete booking
This is handled through an Inngest workflow rather than relying only on a frontend timer.
Stripe Payment Integration
Stripe Checkout is used for payment processing.
When a booking is created, the backend:
1. Validates the selected seats.
2. Calculates the booking amount.
3. Creates a booking record.
4. Creates a Stripe Checkout session.
5. Stores the generated payment link.
6. Associates the booking ID with Stripe session metadata.
7. Starts the payment-status workflow.
After payment, the Stripe webhook:
1. Validates the webhook payload.
2. Reads the payment intent.
3. Finds the corresponding Checkout session.
4. Retrieves the booking ID from metadata.
5. Marks the booking as paid.
6. Emits an Inngest event for confirmation email delivery.
Event-Driven Workflows with Inngest
The project uses Inngest for asynchronous workflows and scheduled tasks.
Implemented workflows include:
User synchronization
Clerk events are used to synchronize users with MongoDB:
clerk/user.created
clerk/user.updated
clerk/user.deleted
Booking expiration
app/checkpayment
        ↓
wait 10 minutes
        ↓
check booking payment
        ↓
release seats if unpaid
Booking confirmation
app/show.booked
        ↓
fetch booking + user + show
        ↓
send confirmation email
Show reminders
A scheduled Inngest function checks upcoming shows and sends reminder emails.
New-show notifications
When a new show is added, the application emits:
app/show.added
and sends notifications to users.
Authentication & Authorization
Authentication is handled using Clerk.
The frontend obtains Clerk session tokens and sends them to the backend using the Authorization header.
Example:
Authorization: Bearer <token>
The backend uses Clerk Express middleware to access the authenticated user's identity.
Admin authorization
Admin show creation uses the user's Clerk private metadata:
privateMetadata.role === "admin"
This is checked by the backend middleware before allowing the protected show-creation route.
API Overview
Shows
| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/api/show/now-playing` | Fetch currently playing movies from TMDB |
| POST | `/api/show/add` | Add a movie show schedule |
| GET | `/api/show/all` | Get available shows |
| GET | `/api/show/:movieId` | Get movie details and upcoming show times |
| DELETE | `/api/show/movie/:movieId` | Delete shows for a movie |
Bookings
| Method | Endpoint | Purpose |
|---|---|---|
| POST | `/api/booking/create` | Create a booking and Stripe Checkout session |
| GET | `/api/booking/seats/:showId` | Get occupied seats |
| POST | `/api/booking/regenerate-payment-link` | Regenerate an unpaid booking payment link |
Users
| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/api/user/bookings` | Get user's bookings |
| POST | `/api/user/update-favorite` | Add/remove a favorite movie |
| GET | `/api/user/favorites` | Get favorite movies |
Admin
| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/api/admin/is-admin` | Check admin status |
| GET | `/api/admin/dashboard` | Get dashboard data |
| GET | `/api/admin/all-shows` | List shows |
| GET | `/api/admin/all-bookings` | List bookings |
Payments
| Method | Endpoint | Purpose |
|---|---|---|
| POST | `/api/stripe` | Receive Stripe webhook events |
Inngest
| Endpoint | Purpose |
|---|---|
| `/api/inngest` | Serve registered Inngest functions |
Project Structure
BookmyshowClone/
│
├── client/
│   ├── public/
│   ├── assets/
│   └── src/
│       ├── components/
│       │   ├── admin/
│       │   └── ...
│       ├── context/
│       ├── lib/
│       ├── pages/
│       │   └── admin/
│       ├── App.jsx
│       └── main.jsx
│
├── server/
│   ├── configs/
│   │   ├── db.js
│   │   └── nodeMailer.js
│   ├── controllers/
│   │   ├── adminController.js
│   │   ├── bookingController.js
│   │   ├── showController.js
│   │   ├── stripeWebhooks.js
│   │   └── userController.js
│   ├── inngest/
│   │   └── index.js
│   ├── middleware/
│   │   └── auth.js
│   ├── models/
│   │   ├── Booking.js
│   │   ├── Movie.js
│   │   ├── Show.js
│   │   └── User.js
│   ├── Routes/
│   │   ├── adminRoutes.js
│   │   ├── bookingRoutes.js
│   │   ├── showRoutes.js
│   │   └── userRoutes.js
│   └── server.js
│
└── README.md
Getting Started
Prerequisites
Make sure the following are installed:
- Node.js 18+
- npm
- MongoDB
- Clerk account
- TMDB API access
- Stripe account
- Inngest account/local development environment
- SMTP credentials for email delivery
Installation
1. Clone the repository
git clone https://github.com/jaydevsingh1611/BookmyshowClone.git
cd BookmyshowClone
2. Install backend dependencies
cd server
npm install
3. Install frontend dependencies
cd ../client
npm install
Environment Variables
Server
Create:
server/.env
Example:
MONGODB_URI=
CLERK_SECRET_KEY=
STRIPE_SECRET_KEY=
STRIPE_WEBHOOK_SECRET=
TMDB_API_KEY=
SMTP_USER=
SMTP_PASS=
SENDER_EMAIL=
Do not commit real credentials or secret keys to GitHub.
Client
Create:
client/.env
Example:
VITE_BASE_URL=http://localhost:3000
VITE_CLERK_PUBLISHABLE_KEY=
VITE_TMDB_IMAGE_BASE_URL=https://image.tmdb.org/t/p/original
VITE_CURRENCY=$
Use the variable names expected by the actual application configuration.
Running Locally
Start the backend
cd server
npm run dev
The backend runs on:
http://localhost:3000
Start the frontend
In another terminal:
cd client
npm run dev
The Vite development server will display the local frontend URL.
Important Engineering Decisions
1. Clerk for authentication
Instead of implementing password storage and session management from scratch, the application delegates authentication to Clerk.
The backend still uses the authenticated Clerk user ID to associate bookings and user data.
2. MongoDB for movie/show/booking data
The application's data is naturally document-oriented:
- Movie metadata contains nested genres and cast information.
- Shows reference movies and maintain occupied-seat state.
- Bookings contain selected seats and payment state.
Mongoose provides schema definitions and population between related documents.
3. Inngest for asynchronous workflows
Tasks such as:
- releasing unpaid seats,
- sending confirmation emails,
- sending show reminders,
- synchronizing Clerk users,
do not need to block the request/response cycle.
They are handled as event-driven or scheduled workflows.
4. Stripe webhooks for payment confirmation
The client redirect is not treated as the source of truth for payment status.
The backend updates the booking after receiving a Stripe payment event.
This separates:
User navigation
from:
Payment confirmation
which is important for payment workflows.
5. TMDB as the movie data source
Movie metadata is retrieved from TMDB rather than manually maintained in the application.
When an admin schedules a movie that is not already stored locally, the backend fetches the movie and credits data and persists it in MongoDB.
Production Hardening / Known Limitations
This project is a learning/portfolio implementation and has areas that should be strengthened before treating it as a production ticketing system.
Atomic seat reservation
The current flow checks seat availability and then updates occupiedSeats in separate database operations.
For high-concurrency production traffic, this should be replaced with an atomic reservation strategy or MongoDB transaction so two simultaneous requests cannot reserve the same seat between the availability check and update.
Admin route protection
The show creation route currently uses the protectAdmin middleware. Other admin endpoints should also enforce backend authorization consistently before production deployment.
Stripe webhook secret
The webhook verification should use a dedicated Stripe webhook signing secret rather than the Stripe API secret key.
Validation and error handling
The API currently returns many application errors as JSON responses without consistently using HTTP status codes.
A production version should introduce centralized validation and error-handling middleware.
Booking consistency
A production booking system should consider transactions, temporary seat holds, expiration semantics, and stronger consistency guarantees around seat inventory and payment state.
Future Improvements
Potential next steps for the project:
- [ ] Add MongoDB transactions for booking/seat reservation
- [ ] Add temporary seat-hold state with explicit expiration
- [ ] Add Redis for high-frequency seat availability reads
- [ ] Add request rate limiting
- [ ] Add centralized API validation
- [ ] Add centralized error-handling middleware
- [ ] Add unit and integration tests
- [ ] Add API documentation with OpenAPI/Swagger
- [ ] Add structured logging
- [ ] Add monitoring and alerting
- [ ] Improve admin authorization across all admin endpoints
- [ ] Add CI/CD pipeline
- [ ] Add automated Stripe webhook tests
- [ ] Add load/concurrency testing for booking flows
What I Learned
This project helped me work through several backend problems beyond basic CRUD:
- Designing REST APIs around multiple related resources
- Managing authentication with an external identity provider
- Modeling movies, shows, bookings, and users with MongoDB/Mongoose
- Handling payment flows with Stripe Checkout and webhooks
- Designing asynchronous workflows with event-driven jobs
- Managing temporary unpaid bookings and seat release
- Integrating third-party APIs such as TMDB
- Connecting frontend state with authenticated backend APIs
- Thinking about concurrency and consistency in a booking system
- Identifying the difference between a working prototype and a production-ready system
Demo
Add the deployed frontend URL here once you have a stable public deployment.
Live Demo: <https://bookmyshowclone-1-k0xi.onrender.com/>
Author
Jaydev Singh Chahar
B.Tech — NIT Patna
- GitHub: jaydevsingh1611
- Email: jaydevsinghchahar1611@gmail.com
License
This project is intended for educational and portfolio purposes.