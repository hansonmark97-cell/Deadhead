# Deadhead

Deadhead combines on-demand LTL brokerage with deadhead route tracking to match partial loads to trucks already driving empty return miles.

## Core MVP

### Carrier app
- Post route: origin, destination, departure time, available space/weight.
- Set deviation tolerance (max detour miles).
- Share live GPS after accepting a load.

### Shipper app
- Post load: dimensions, weight, pallet count/class, pickup/delivery windows.
- Get instant route matches.
- Use digital BOL/POD at pickup and delivery.

## Matching engine (PostgreSQL + PostGIS)

Use PostGIS to store:
- `deadhead_routes.route_path` as `LINESTRING`
- `active_loads.pickup_location` and `dropoff_location` as `POINT`

Then match pending loads to searching routes when:
- load space/weight fits the truck
- pickup and dropoff are both within `deviation_miles` of the route path (`ST_DWithin`)

## Automated money flow (Stripe Connect)

1. Shipper pays full amount at booking; funds are held in platform escrow.
2. Delivery is verified with GPS + POD.
3. Backend releases payout:
   - Standard: 9% platform fee, 91% driver payout
   - Instant: 15% platform fee, 85% driver payout (6% premium for faster payout)
4. Stripe handles transfer and instant payout rails automatically.

## Suggested backend execution sequence

1. Driver onboarding via Stripe Connect Express (`acct_xxx` stored per driver).
2. Match webhook locks load + creates escrowed PaymentIntent.
3. Delivery verification webhook computes fee tier (standard vs instant).
4. Transfer to connected account, and optional instant payout if selected.

## Security and implementation notes

- Never hardcode API keys in source code; use environment variables.
- Verify webhook signatures for both database events and Stripe webhooks.
- Store all monetary amounts in cents to avoid float precision errors.
- Keep idempotency keys on payment and payout operations.

## Initial architecture

```text
[Shipper App] -- post load --> [Central DB + Geo-Match]
                                    |
[Driver App] -- post route -->      |
                                    v
                             [Escrow + Payout Engine]
                               /                  \
                       (9%/15% platform fee)   (driver payout)
```
