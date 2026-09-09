# 规格模式（Specification Pattern）

规格模式（Specification Pattern），是一种用于封装业务规则或条件的结构型设计模式，使这些规则能够复用、组合和测试。它主要处理复杂的过滤逻辑或验证规则，以及这些规则可能会随着时间的推移发生变化，隔离这些规则，以达到单一职责原则。

这个模式可以灵活地对业务逻辑进行定制，只需将业务规则或条件（主要是条件）以某种关系（与、或、非）进行组合。

### 使用场景

看到这个模式，还是在讲DDD设计的文章中提到的，将复杂条件封装为规格，然后调用时组合各个规则，代码清晰明了。可以为领域模型的实体、值对象、领域服务等等简化复杂逻辑的代码。

- 验证对象：确保对象在处理前满足特定条件。

- 查询系统：基于各种动态条件从集合中过滤对象。
- 指定在创建新对象的时候必须要满足某种业务要求。

### 标准的示例代码

1，规格的基本工具，定义及与或非：

```java
//抽象规格
interface ISpecification {
	//候选者是否满足条件
	boolean isSatisfiedBy (Object candidate) ;
	//and操作
	ISpecification and (ISpecification spec);
	//or操作
	ISpecification or (ISpecification spec);
	//not操作
	ISpecification not ();
}

//组合规格
abstract class CompositeSpecification implements ISpecification {
	//是否满足条件由子类实现
	public abstract boolean isSatisfiedBy (Object candidate) ;
	//and操作
	public ISpecification and (ISpecification spec) {
		return new AndSpecification(this, spec);
	}
	//or操作
	public ISpecification or(ISpecification spec) {
		return new OrSpecification(this, spec);
	}
	//not操作
	public ISpecification not() {
		return new NotSpecification(this);
	}
}

//与规格
class AndSpecification extends CompositeSpecification {
	//传递两个规格书进行and操作
	private ISpecification left;
	private ISpecification right;

	public AndSpecification(ISpecification left, ISpecification right) {
		this.left = left;
		this.right = right;
	}
	
	//进行and运算
	public boolean isSatisfiedBy(Object candidate) {
		return left.isSatisfiedBy(candidate) && right.isSatisfiedBy(candidate);
	}
}

//或规格
class OrSpecification extends CompositeSpecification {
	//传递两个规格书进行or操作
	private ISpecification left;
	private ISpecification right;

	public OrSpecification(ISpecification left, ISpecification right) {
		this.left= left;
		this.right = right;
	}

	//进行or运算
	public boolean isSatisfiedBy(Object candidate) {
		return left.isSatisfiedBy(candidate) || right.isSatisfiedBy(candidate);
	}
}
//非规格
class NotSpecification extends CompositeSpecification {
	//传递一个规格书进行非操作
	private ISpecification spec;

	public NotSpecification(ISpecification spec) {
		this.spec = spec;
	}

	//进行not运算
	public boolean isSatisfiedBy(Object candidate) {
		return !spec.isSatisfiedBy(candidate);
	}
}
```

2，组织自己的业务规格：

```
//业务规格
class BizSpecification extends CompositeSpecification {
	//基准对象，如姓名等，也可以是int等类型
	private String obj;
	public BizSpecification(String obj) {
		this.obj = obj;
	}
	//判断是否满足要求
	public boolean isSatisfiedBy(Object candidate){
		//根据基准对象判断是否符合
		return true;
	}
}
//客户端调用
public class Client {

    public static void main(String[] args) {
        //待分析的对象
        List<Object> list = new ArrayList<Object>();
        //定义两个业务规格
        ISpecification spec1 = new BizSpecification("a");
        ISpecification spec2 = new BizSpecification("b");
        //规格调用
        for (Object o : list) {
            if(spec1.and(spec2).isSatisfiedBy(o)){  //如果o满足spec1 && spec2
                System.out.println(o);
            }
        }
    }
}
```

### 另一个简单示例代码

假设需要根据价格、类别和库存等各种条件来过滤商品。相比硬编码这些规则，使用规格模式来处理就清晰多了。

```java
// 规格定义
public interface Specification<T> {
    boolean isSatisfiedBy(T candidate);
}

// 实现价格条件规格
public class PriceSpecification implements Specification<Product> {
    private double maxPrice;

    public PriceSpecification(double maxPrice) {
        this.maxPrice = maxPrice;
    }

    @Override
    public boolean isSatisfiedBy(Product product) {
        return product.getPrice() <= maxPrice;
    }
}
// 实现品类条件规格
public class CategorySpecification implements Specification<Product> {
    private String category;

    public CategorySpecification(String category) {
        this.category = category;
    }

    @Override
    public boolean isSatisfiedBy(Product product) {
        return product.getCategory().equalsIgnoreCase(category);
    }
}

// 实现And规格
public class AndSpecification<T> implements Specification<T> {
    private Specification<T> spec1;
    private Specification<T> spec2;

    public AndSpecification(Specification<T> spec1, Specification<T> spec2) {
        this.spec1 = spec1;
        this.spec2 = spec2;
    }

    @Override
    public boolean isSatisfiedBy(T candidate) {
        return spec1.isSatisfiedBy(candidate) && spec2.isSatisfiedBy(candidate);
    }
}

// 客户端使用规格，1，配置条件（价格<=100 and 品类=Book）
Specification<Product> spec = new AndSpecification<>(
    new PriceSpecification(100.0),
    new CategorySpecification("Book")
);
// 2，判断product
boolean isEligible = spec.isSatisfiedBy(product);
```

