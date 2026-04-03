// 核心SVG操作库，模拟d3.js链式API
class D3Like {
  constructor(elements) {
    this._elements = elements; // 存储当前选中的DOM元素数组
  }

  // 核心选择方法
  static select(selector) {
    const element = typeof selector === 'string' 
      ? document.querySelector(selector) 
      : selector;
    return new D3Like(element ? [element] : []);
  }

  static selectAll(selector) {
    const elements = typeof selector === 'string'
      ? document.querySelectorAll(selector)
      : selector;
    return new D3Like(Array.from(elements));
  }

  // 属性操作
  attr(name, value) {
    if (value === undefined) {
      return this._elements[0]?.getAttribute(name);
    }
    this._elements.forEach(el => {
      if (value === null) {
        el.removeAttribute(name);
      } else {
        el.setAttribute(name, value);
      }
    });
    return this;
  }

  // 样式操作
  style(name, value, priority) {
    if (value === undefined) {
      return this._elements[0]?.style[name];
    }
    this._elements.forEach(el => {
      el.style.setProperty(name, value, priority || '');
    });
    return this;
  }

  // 文本内容
  text(content) {
    if (content === undefined) {
      return this._elements[0]?.textContent;
    }
    this._elements.forEach(el => {
      el.textContent = content;
    });
    return this;
  }

  // HTML内容
  html(content) {
    if (content === undefined) {
      return this._elements[0]?.innerHTML;
    }
    this._elements.forEach(el => {
      el.innerHTML = content;
    });
    return this;
  }

  // 添加/移除类
  classed(className, value) {
    if (value === undefined) {
      return this._elements[0]?.classList.contains(className);
    }
    this._elements.forEach(el => {
      if (value) {
        el.classList.add(className);
      } else {
        el.classList.remove(className);
      }
    });
    return this;
  }

  // 添加/移除类（别名）
  classedAdd(className) {
    return this.classed(className, true);
  }

  classedRemove(className) {
    return this.classed(className, false);
  }

  // 事件监听
  on(event, handler, options) {
    this._elements.forEach(el => {
      if (handler === null) {
        el.removeEventListener(event, this._eventHandlers?.get(el));
      } else {
        if (!this._eventHandlers) this._eventHandlers = new Map();
        this._eventHandlers.set(el, handler);
        el.addEventListener(event, handler, options);
      }
    });
    return this;
  }

  // 添加子元素
  append(type) {
    const newElements = [];
    this._elements.forEach(parent => {
      const child = document.createElementNS('http://www.w3.org/2000/svg', type);
      parent.appendChild(child);
      newElements.push(child);
    });
    return new D3Like(newElements);
  }

  // 插入元素（在指定元素前）
  insert(type, before) {
    const newElements = [];
    this._elements.forEach(parent => {
      const child = document.createElementNS('http://www.w3.org/2000/svg', type);
      const refNode = parent.querySelector(before);
      parent.insertBefore(child, refNode);
      newElements.push(child);
    });
    return new D3Like(newElements);
  }

  // 移除元素
  remove() {
    this._elements.forEach(el => el.remove());
    return new D3Like([]);
  }

  // 数据绑定
  data(dataArray, keyFunction) {
    if (dataArray === undefined) {
      return this._elements.map(el => el.__data__);
    }

    // 处理更新、退出、进入选择集
    const update = [];
    const exit = [];
    const enter = [];

    if (keyFunction) {
      // 使用key函数进行更精确的数据绑定
      const dataMap = new Map();
      dataArray.forEach((d, i) => {
        const key = keyFunction(d, i);
        dataMap.set(key, d);
      });

      this._elements.forEach(el => {
        const elKey = el.__data__ ? keyFunction(el.__data__, 0) : null;
        if (elKey !== null && dataMap.has(elKey)) {
          update.push(el);
          el.__data__ = dataMap.get(elKey);
          dataMap.delete(elKey);
        } else {
          exit.push(el);
        }
      });

      // 剩余的数据进入
      dataMap.forEach((d) => {
        enter.push(d);
      });
    } else {
      // 简单索引绑定
      this._elements.forEach((el, i) => {
        if (i < dataArray.length) {
          update.push(el);
          el.__data__ = dataArray[i];
        } else {
          exit.push(el);
        }
      });

      for (let i = this._elements.length; i < dataArray.length; i++) {
        enter.push(dataArray[i]);
      }
    }

    // 创建进入选择集（虚拟元素）
    const enterSelection = {
      append: (type) => {
        const newElements = [];
        enter.forEach(data => {
          const el = document.createElementNS('http://www.w3.org/2000/svg', type);
          el.__data__ = data;
          // 这里需要父容器，暂时无法获取，需要外部传入
          newElements.push(el);
        });
        return new D3Like(newElements);
      }
    };

    // 返回标准d3风格的选择集对象
    return {
      enter: () => enterSelection,
      exit: () => new D3Like(exit).remove(),
      _groups: [update],
      _parents: [this._elements[0]?.parentNode]
    };
  }

  // 获取/设置数据
  datum(value) {
    if (value === undefined) {
      return this._elements[0]?.__data__;
    }
    this._elements.forEach(el => {
      el.__data__ = value;
    });
    return this;
  }

  // 过渡动画（简化版）
  transition(duration = 250, ease = 'ease') {
    const transitions = [];
    this._elements.forEach(el => {
      const transition = {
        attr: (name, value) => {
          const start = parseFloat(el.getAttribute(name)) || 0;
          const end = parseFloat(value);
          const startTime = performance.now();
          const animate = (currentTime) => {
            const elapsed = currentTime - startTime;
            const progress = Math.min(1, elapsed / duration);
            const eased = this._ease(progress, ease);
            const currentValue = start + (end - start) * eased;
            el.setAttribute(name, currentValue);
            if (progress < 1) requestAnimationFrame(animate);
          };
          requestAnimationFrame(animate);
          return transition;
        },
        style: (name, value) => {
          const start = parseFloat(el.style[name]) || 0;
          const end = parseFloat(value);
          const startTime = performance.now();
          const animate = (currentTime) => {
            const elapsed = currentTime - startTime;
            const progress = Math.min(1, elapsed / duration);
            const eased = this._ease(progress, ease);
            const currentValue = start + (end - start) * eased;
            el.style[name] = currentValue;
            if (progress < 1) requestAnimationFrame(animate);
          };
          requestAnimationFrame(animate);
          return transition;
        },
        duration: (ms) => {
          // 简化处理，实际应该返回新transition
          return transition;
        }
      };
      transitions.push(transition);
    });
    return transitions[0] || {};
  }

  _ease(t, ease) {
    switch(ease) {
      case 'linear': return t;
      case 'ease-in': return t * t;
      case 'ease-out': return t * (2 - t);
      case 'ease': default: return t < 0.5 ? 4 * t * t * t : 1 - Math.pow(-2 * t + 2, 3) / 2;
    }
  }

  // 选择集遍历
  each(callback) {
    this._elements.forEach((el, i) => {
      callback.call(el, el.__data__, i);
    });
    return this;
  }

  // 过滤选择集
  filter(selector) {
    const filtered = this._elements.filter(el => {
      if (typeof selector === 'function') {
        return selector.call(el, el.__data__, this._elements.indexOf(el));
      } else {
        return el.matches(selector);
      }
    });
    return new D3Like(filtered);
  }

  // 获取节点数量
  size() {
    return this._elements.length;
  }

  // 获取原生节点
  node() {
    return this._elements[0];
  }

  nodes() {
    return this._elements;
  }

  // SVG图形创建辅助方法
  static createSVG(width, height) {
    const svg = document.createElementNS('http://www.w3.org/2000/svg', 'svg');
    svg.setAttribute('width', width);
    svg.setAttribute('height', height);
    return svg;
  }
}

// 导出全局d3对象
window.d3 = {
  select: D3Like.select,
  selectAll: D3Like.selectAll,
  // 添加常用比例尺和辅助函数（可选）
  scaleLinear: () => ({
    domain: () => {},
    range: () => {},
    interpolate: () => {}
  }),
  max: (arr, fn) => Math.max(...arr.map(fn)),
  min: (arr, fn) => Math.min(...arr.map(fn))
};