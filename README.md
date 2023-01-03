Домен
======

Эта библиотека позволяет вам сконцентрироваться на самом важном в вашем приложении: на домене.
Согласно книге Эрика Эвана «Дизайн, управляемый доменом», ваш домен должен состоять из агрегатов.
Каждый агрегат состоит из AggregateRoot и, возможно, некоторого ObjectValue.

Эта библиотека ориентирована на AggregateRoot и предоставляет помощника для управления событиями, происходящими с вашим агрегатом.

Применение
-----

Пример лучше объяснит назначение этой библиотеки:

```php
use PhpDDD\Domain\AbstractAggregateRoot;

class PurchaseOrder extends AbstractAggregateRoot
{
    /**
     * @var mixed
     */
    private $customerId;

    /**
     * Try to create a new PurchaseOrder for a client.
     */
    public function __construct(Customer $customer)
    {
        // Tests that some business rules are followed.
        if (false === $customer->isActive()) {
            throw new MyDomainException('The customer need to be active.');
        }
        
        // ...
        // Since every rules are validated, we can simulate the creation of the Event associated
        // This allows us to have less redundant code
        
        $this->apply(new PurchaseOrderCreated($customer, new DateTime()));
        
        // This is equivalent as
        // $this->customerId = $customer->getId();
        // $this->events[] = new PurchaseOrderCreated($customer, new DateTime());
        // but we are not allowing to add event directly.
    }
    
    /**
     * The apply() method will automatically call this method.
     * Since it's an event you should never do some tests in this method.
     * Try to think that an Event is something that happened in the past.
     * You can not modify what happened. The only thing that you can do is create another event to compensate.
     */
    protected function applyPurchaseOrderCreated(PurchaseOrderCreated $event)
    {
        $this->customerId = $event->getCustomer()->getId();
    }
}
```

Что хорошего в этом подходе, так это то, что вы можете затем прослушивать все события, созданные вашим AggregateRoot, и выполнять некоторые дополнительные действия, которые на самом деле не имеют отношения к вашему бизнес-домену, например отправлять электронное письмо.

```php
use PhpDDD\Domain\Event\Listener\AbstractEventListener;

class SendEmailOnPurchaseOrderCreated extends AbstractEventListener
{
    private $mailer;
    
    public function __construct($mailer)
    {
        $this->mailer = $mailer;
    }
    
    /**
     * {@inheritDoc}
     */
    public function getSupportedEventClassName()
    {
        return 'PurchaseOrderCreated'; // Should be the fully qualified class name of the event
    }
    
    /**
     * {@inheritDoc}
     */
    public function handle(EventInterface $event)
    {
        $this->mailer->send('to@you.com', 'Purchase order created for customer #' . $event->getCustomer()->getId());
    }
}
```

Чтобы ваш проект знал, что EventListener привязан к событию, вы должны использовать EventBus:

```php
// first the locator
$locator = new \PhpDDD\Domain\Event\Listener\Locator\EventListenerLocator();
$locator->register('PurchaseOrderCreated', new SendEmailOnPurchaseOrderCreated(/* $mailer */));

// then the EventBus
$bus = new \PhpDDD\Domain\Event\Bus\EventBus($locator);

// do what you need to do on your Domain
$aggregateRoot = new PurchaseOrder(new Customer(1));

// Then apply EventListeners
$events = $aggregateRoot->pullEvents(); // this will clear the list of event in your AggregateRoot so an Event is trigger only once

// You can have more than one event at a time.
foreach($events as $event) {
    $bus->publish($event);
}
```
