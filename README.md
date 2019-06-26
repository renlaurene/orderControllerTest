# orderControllerTest
how to handle the online order system


package npu.orderapp.tests;

import static org.junit.Assert.assertEquals;
import static org.junit.Assert.assertNotNull;
import static org.junit.Assert.assertTrue;
import static org.junit.Assert.fail;

import java.util.Calendar;
import java.util.Date;
import java.util.GregorianCalendar;

import npu.orderapp.domain.Order;
import npu.orderapp.domain.Orders;

import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.http.HttpStatus;
import org.springframework.test.context.ContextConfiguration;
import org.springframework.test.context.junit4.SpringJUnit4ClassRunner;
import org.springframework.web.client.HttpClientErrorException;
import org.springframework.web.client.RestTemplate;


@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations = { "/test-context.xml"})
public class OrderControllerTest {
	private static final String ORDER_URL = "http://localhost:8080/restorders/orders";
	private static final String ORDER_ID_URL = ORDER_URL + "/{id}";
	@Autowired
	private RestTemplate restTemplate;
	
	@Test
	public void testHaveRestTemplate() {
		assertNotNull(restTemplate);
	}
	
	@Test
	public void testPostOrder() {
		Date orderDate;
		Order retrievedOrder;
		Order newOrder = new Order("ZG4568", 250.75F);
		
		assertNotNull(restTemplate);
		retrievedOrder = restTemplate.postForObject(ORDER_URL, newOrder,Order.class);
		
		assertNotNull(retrievedOrder);
		/* Some things are different in retrieved order -- its date and id should now be set */
		assertTrue(newOrder.dataEqualWithoutDate(retrievedOrder));
		
		/* Make sure the date got set correctly */
		orderDate = retrievedOrder.getDate();
		assertNotNull(orderDate);
		GregorianCalendar curDate = new GregorianCalendar();
		int curDay = curDate.get(GregorianCalendar.DAY_OF_YEAR);
		Calendar cmpDate = new GregorianCalendar(); 
		cmpDate.setTime(orderDate);
		int orderDay = cmpDate.get(Calendar.DAY_OF_YEAR);
		assertEquals(curDay, orderDay);
		
		/* The Id should have been set to something greater than 0 */
		assertTrue(retrievedOrder.getId() > 0);
	}
	
	@Test
	public void getOrdersTest() throws Exception {
		Orders initialOrders = restTemplate.getForObject(ORDER_URL, Orders.class);
		assertNotNull(initialOrders);
		int initialSize = initialOrders.size();
		
		/* We'll add two orders and then verify that we get back a size that has two more orders */
		Order newOrder = new Order("ZG4568", 250.75F);
		restTemplate.postForObject(ORDER_URL, newOrder,Order.class);
		
		newOrder = new Order("JY3872", 58.98F);
		restTemplate.postForObject(ORDER_URL, newOrder,Order.class);
		
		Orders orders = restTemplate.getForObject(ORDER_URL, Orders.class);
		assertNotNull(orders);
		int newSize = orders.size();
		/* We added two new orders */
		assertEquals(initialSize+2, newSize);
	}
	
	@Test
	public void getOrderTest() throws Exception {
		Order createdOrder, retrievedOrder;
		Order newOrder = new Order("ZG4568", 250.75F);
		
		createdOrder = restTemplate.postForObject(ORDER_URL, newOrder,Order.class);
		assertNotNull(createdOrder);
		
		int newId = createdOrder.getId();
		assertTrue(newId > 0);
		
		/* Now for our real test -- lookup the order */
		retrievedOrder = restTemplate.getForObject(ORDER_ID_URL, Order.class, newId);
		
		assertEquals(createdOrder, retrievedOrder);
	}
	
	@Test
	public void deleteOrderTest() throws Exception {
		Order createdOrder, retrievedOrder;
		Order newOrder = new Order("ZG4568", 250.75F);
		
		createdOrder = restTemplate.postForObject(ORDER_URL, newOrder,Order.class);
		assertNotNull(createdOrder);
		
		int newId = createdOrder.getId();
		assertTrue(newId > 0);
		
		retrievedOrder = restTemplate.getForObject(ORDER_ID_URL, Order.class, newId);
		/* Before we delete the order we need to be sure the order is there */
		assertNotNull(retrievedOrder);
		assertEquals(createdOrder, retrievedOrder);
		
		restTemplate.delete(ORDER_URL +"/{id}", newId);
		
		try {
			retrievedOrder = restTemplate.getForObject(ORDER_ID_URL, Order.class, newId);
			/* The order should have been removed */
			fail();
		} catch (HttpClientErrorException ex) {
			HttpStatus status = ex.getStatusCode();
			/* We expect the order was not found.  Otherwise this test failed.  */
			assertEquals(status,HttpStatus.NOT_FOUND);
		}
	}
	
	@Test
	public void changeOrderTest() throws Exception {
		final float DELTA = 1E-10F;
		float newAmount = 100.00F;
		Order createdOrder, initialOrder, modifiedOrder;
		Order newOrder = new Order("ZG4568", 250.75F);
		
		/*  Create an order and then change its amount.  */
		createdOrder = restTemplate.postForObject(ORDER_URL, newOrder,Order.class);
		assertNotNull(createdOrder);
		
		int newId = createdOrder.getId();
		assertTrue(newId > 0);
		initialOrder = restTemplate.getForObject(ORDER_ID_URL, Order.class, newId);
		assertNotNull(initialOrder);
		
		initialOrder.setAmount(100.00F);
		restTemplate.put(ORDER_ID_URL, initialOrder, newId);
		
		modifiedOrder = restTemplate.getForObject(ORDER_ID_URL, Order.class, newId);
		assertEquals(modifiedOrder.getId(), initialOrder.getId());
		assertEquals(modifiedOrder.getAmount(), newAmount, DELTA);
	}
}
