# LinqCV Auto Order Recovery System

Note: This is a writing sample copied from a private repo and included here for portfolio review.

This system automatically handles stuck or failed orders to ensure users never lose their payments or credits.

## How It Works

### 1. Login-Based Recovery (Always Active)
- When any user logs in, the system automatically checks for stuck orders for that user
- Attempts to retry processing failed orders
- Runs in the background without affecting login speed

### 2. Background Auto Checker (Development)
- Runs continuously in the background during development
- Checks for stuck orders every 5 minutes
- Handles orders that may have timed out during webhook processing

### 3. Manual Management Command
- `python manage.py retry_stuck_orders` - Check and retry stuck orders
- `--max-age-hours=1` - Only process orders older than 1 hour
- `--dry-run` - See what would be done without making changes

## For Development

### Start the Auto Checker
```bash
cd /path/to/LinqCV
./manage_checker.sh start
```

### Check Status
```bash
./manage_checker.sh status
```

### Stop the Auto Checker
```bash
./manage_checker.sh stop
```

### Restart the Auto Checker
```bash
./manage_checker.sh restart
```

### Manual Check
```bash
./check_stuck_orders.sh
```

## For Production

Set up a cron job to run every 30 minutes:
```bash
*/30 * * * * cd /path/to/your/project/backend && source venv/bin/activate && python manage.py retry_stuck_orders --max-age-hours=1
```

## What Happens When Orders Fail

### Paid Orders
- Automatically refunded via Stripe
- Order marked as "failed" with error message
- User sees error in dashboard

### Free Credit Orders
- Credit automatically restored to user's account
- Order marked as "failed"
- User sees error in dashboard

## Files

- `auto_order_checker.py` - Background service script
- `manage_checker.sh` - Start/stop/status management script
- `check_stuck_orders.sh` - Simple manual check script
- `backend/apps/resumes/management/commands/retry_stuck_orders.py` - Django management command
- `backend/apps/users/signals.py` - Login-based recovery
- `backend/apps/users/models.py` - User model methods for order recovery

## Monitoring

Check the logs:
```bash
tail -f auto_checker.log
```

The system will log all activity including successful retries, refunds, and any errors encountered.
