# hellocoffee_mv

### Webservice

```swift
import Foundation

enum NetworkError: Error {
    case badRequest
    case decodingError
    case badUrl
}

class Webservice {
    
    private var baseURL: URL
    
    init(baseURL: URL) {
        self.baseURL = baseURL
    }
    
    func updateOrder(_ order: Order) async throws -> Order {
        
        guard let orderId = order.id else {
            throw NetworkError.badRequest
        }
        
        guard let url = URL(string: Endpoints.updateOrder(orderId).path, relativeTo: baseURL) else {
            throw NetworkError.badUrl
        }
       
        var request = URLRequest(url: url)
        request.httpMethod = "PUT"
        request.addValue("application/json", forHTTPHeaderField: "Content-Type")
        request.httpBody = try JSONEncoder().encode(order)
        
        let (data, response) = try await URLSession.shared.data(for: request)
        
        guard let httpResponse = response as? HTTPURLResponse,
              httpResponse.statusCode == 200 else {
            throw NetworkError.badRequest
        }
        
        guard let updatedOrder = try? JSONDecoder().decode(Order.self, from: data) else {
            throw NetworkError.decodingError
        }
        
        return updatedOrder
        
    }
    
    func deleteOrder(orderId: Int) async throws -> Order {
        
        guard let url = URL(string: Endpoints.deleteOrder(orderId).path, relativeTo: baseURL) else {
            throw NetworkError.badUrl
        }
        
        var request = URLRequest(url: url)
        request.httpMethod = "DELETE"
        
        let (data, response) = try await URLSession.shared.data(for: request)
        guard let httpResponse = response as? HTTPURLResponse,
              httpResponse.statusCode == 200 else {
            throw NetworkError.badRequest
        }
        
        guard let deletedOrder = try? JSONDecoder().decode(Order.self, from: data) else {
            throw NetworkError.decodingError
        }
        
        return deletedOrder
    }
    
    func placeOrder(order: Order) async throws -> Order {
        
        guard let url = URL(string: Endpoints.placeOrder.path, relativeTo: baseURL) else {
                   throw NetworkError.badUrl
        }
        
        var request = URLRequest(url: url)
        request.httpMethod = "POST"
        request.addValue("application/json", forHTTPHeaderField: "Content-Type")
        request.httpBody = try JSONEncoder().encode(order)
        
        let (data, response) = try await URLSession.shared.data(for: request)
        
        guard let httpResponse = response as? HTTPURLResponse,
                     httpResponse.statusCode == 200 else {
                   throw NetworkError.badRequest
        }
        
        guard let newOrder = try? JSONDecoder().decode(Order.self, from: data) else {
            throw NetworkError.decodingError
        }
        
        return newOrder
        
    }
    
    func getOrders() async throws -> [Order] {
        
        guard let url = URL(string: Endpoints.allOrders.path, relativeTo: baseURL) else {
            throw NetworkError.badUrl
        }
        
        let (data, response) = try await URLSession.shared.data(from: url)
        guard let httpResponse = response as? HTTPURLResponse,
              httpResponse.statusCode == 200 else {
            throw NetworkError.badRequest
        }
        
        guard let orders = try? JSONDecoder().decode([Order].self, from: data) else {
            throw NetworkError.decodingError
        }
        
        return orders
    }
    
}

```

### Coffee Model
```swift
import Foundation

@MainActor
class CoffeeModel: ObservableObject {
    
    let webservice: Webservice
    @Published private(set) var orders: [Order] = []
    
    init(webservice: Webservice) {
        self.webservice = webservice
    }
    
    func orderById(_ id: Int) -> Order? {
        guard let index = orders.firstIndex(where: { $0.id == id }) else {
            return nil
        }
        
        return orders[index]
    }
    
    func populateOrders() async throws {
        orders = try await webservice.getOrders()
    }
    
    func placeOrder(_ order: Order) async throws {
        let newOrder = try await webservice.placeOrder(order: order)
        orders.append(newOrder)
    }
    
    func deleteOrder(_ orderId: Int) async throws {
        let deletedOrder = try await webservice.deleteOrder(orderId: orderId)
        orders = orders.filter { $0.id != deletedOrder.id }
    }
    
    func updateOrder(_ order: Order) async throws {
        let updatedOrder = try await webservice.updateOrder(order)
        guard let index = orders.firstIndex(where: { $0.id == updatedOrder.id }) else {
            throw CoffeeOrderError.invalidOrderId
        }
        orders[index] = updatedOrder
    }
    
}

```
