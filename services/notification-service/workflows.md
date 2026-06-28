# NotificationService Workflows

## In-App Notification

```text
Domain service
  -> notification event/command
  -> NotificationService creates notification
  -> SignalR sends ReceiveNotification
  -> user marks notification read
```

## Email Notification

```text
Domain service
  -> notification event/command
  -> NotificationService resolves user reference and template
  -> email provider send
  -> delivery attempt recorded
```

## OTP Sign-In Email Delivery

```text
IdentityService
  -> UserOtpChallengeRequestedIntegrationEvent
  -> NotificationService selects the OTP sign-in template for Purpose = SignIn
  -> email provider send
  -> delivery attempt recorded
```

## Fulfillment Continuation

```text
OrderService FulfillmentSaga
  -> NotifySellerNewOrder / NotifyBuyerOrderConfirmed / NotifyBuyerOrderCancelled
  -> NotificationService records delivery
  -> SellerNewOrderNotifiedIntegrationEvent or BuyerNotifiedIntegrationEvent
```

## Preference Handling

```text
User updates preferences
  -> channel preference and/or event-group preference saved
  -> later delivery checks preferences before sending
```

## Workflow Rules

- Delivery may be skipped by preference, but the source-domain event remains true.
- Failed delivery should be observable through delivery attempts.
- Realtime push complements persistence; it must not be the only record.
- OTP challenge delivery follows the published challenge purpose; v1 supports `SignIn` only.
- Fulfillment continuation publishes remain MassTransit consume-context orchestration messages.
