# Nên thay thế dần cho for each truyền thống
## Không nên viết quá phức tạm
```java
List<CustomerSummary> summaries = orders.stream()
    .filter(order -> order.getStatus() == Status.COMPLETED && order.getAmount() > 100)
    .collect(Collectors.groupingBy(Order::getCustomerId,
        Collectors.summingDouble(Order::getAmount)))
    .entrySet().stream()
    .filter(entry -> entry.getValue() > 1000)
    .sorted(Map.Entry.<Long, Double>comparingByValue().reversed())
    .map(entry -> new CustomerSummary(entry.getKey(), entry.getValue()))
    .collect(Collectors.toList());
```

## Tách ra cho dễ đọc
```java
// 1. Lọc đơn hàng đủ điều kiện
List<Order> highValueCompletedOrders = orders.stream()
    .filter(order -> order.getStatus() == Status.COMPLETED)
    .filter(order -> order.getAmount() > 100)
    .toList();

// 2. Gom nhóm và tính tổng tiền theo customer
Map<Long, Double> totalByCustomer = highValueCompletedOrders.stream()
    .collect(Collectors.groupingBy(
        Order::getCustomerId,
        Collectors.summingDouble(Order::getAmount)
    ));

// 3. Lọc ra khách hàng có tổng chi tiêu > 1000
List<Map.Entry<Long, Double>> highSpendingCustomers = totalByCustomer.entrySet().stream()
    .filter(entry -> entry.getValue() > 1000)
    .sorted(Map.Entry.<Long, Double>comparingByValue().reversed())
    .toList();

// 4. Ánh xạ thành CustomerSummary
List<CustomerSummary> summaries = highSpendingCustomers.stream()
    .map(entry -> new CustomerSummary(entry.getKey(), entry.getValue()))
    .toList();
```

## Ghi chú các bước
```java
Stream<Person> personStream = people.stream();

// Bước 1: Lọc người đủ tuổi
Stream<Person> adults = personStream.filter(p -> p.getAge() > 18);

// Bước 2: Lấy tên
Stream<String> names = adults.map(Person::getName);

// Bước 3: Loại trùng
Stream<String> uniqueNames = names.distinct();

// Bước 4: Sắp xếp
Stream<String> sortedNames = uniqueNames.sorted();

// Bước 5: Giới hạn số lượng
List<String> result = sortedNames.limit(5).collect(Collectors.toList());
```

## Đặt tên gợi nhớ
```java
Stream<Person> personStream = people.stream();
Stream<Person> adults = personStream.filter(p -> p.getAge() > 18);
Stream<String> adultNames = adults.map(Person::getName);
Stream<String> distinctNames = adultNames.distinct();
Stream<String> sortedTopNames = distinctNames.sorted().limit(5);

List<String> result = sortedTopNames.collect(Collectors.toList());
```
