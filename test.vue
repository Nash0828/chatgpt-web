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

  // call方法：调用一个函数，传入当前选择集
  call(fn, ...args) {
    fn(this, ...args);
    return this;
  }

  // 属性操作
  attr(name, value) {
    if (value === undefined) {
      return this._elements[0]?.getAttribute(name);
    }
    this._elements.forEach(el => {
      if (value === null) {
        el.removeAttribute(name);
      } else if (typeof value === 'function') {
        el.setAttribute(name, value.call(el, el.__data__, this._elements.indexOf(el)));
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
      if (typeof value === 'function') {
        el.style.setProperty(name, value.call(el, el.__data__, this._elements.indexOf(el)), priority || '');
      } else {
        el.style.setProperty(name, value, priority || '');
      }
    });
    return this;
  }

  // 文本内容
  text(content) {
    if (content === undefined) {
      return this._elements[0]?.textContent;
    }
    this._elements.forEach(el => {
      if (typeof content === 'function') {
        el.textContent = content.call(el, el.__data__, this._elements.indexOf(el));
      } else {
        el.textContent = content;
      }
    });
    return this;
  }

  // HTML内容
  html(content) {
    if (content === undefined) {
      return this._elements[0]?.innerHTML;
    }
    this._elements.forEach(el => {
      if (typeof content === 'function') {
        el.innerHTML = content.call(el, el.__data__, this._elements.indexOf(el));
      } else {
        el.innerHTML = content;
      }
    });
    return this;
  }

  // 添加/移除类
  classed(className, value) {
    if (value === undefined) {
      return this._elements[0]?.classList.contains(className);
    }
    this._elements.forEach(el => {
      const shouldAdd = typeof value === 'function' 
        ? value.call(el, el.__data__, this._elements.indexOf(el))
        : value;
      if (shouldAdd) {
        el.classList.add(className);
      } else {
        el.classList.remove(className);
      }
    });
    return this;
  }

  // 事件监听
  on(event, handler, options) {
    this._elements.forEach(el => {
      if (handler === null) {
        const stored = this._eventHandlers?.get(el);
        if (stored) {
          el.removeEventListener(event, stored);
          this._eventHandlers.delete(el);
        }
      } else {
        if (!this._eventHandlers) this._eventHandlers = new Map();
        this._eventHandlers.set(el, handler);
        el.addEventListener(event, handler, options);
      }
    });
    return this;
  }

  // drag方法：实现拖拽功能
  drag(dragHandler) {
    // 如果没有传入handler，返回默认的drag行为
    if (!dragHandler) {
      dragHandler = {
        onDragStart: null,
        onDrag: null,
        onDragEnd: null
      };
    }

    this._elements.forEach(el => {
      let startX = 0, startY = 0;
      let initialX = 0, initialY = 0;
      let isDragging = false;

      const onMouseDown = (e) => {
        e.preventDefault();
        isDragging = true;
        startX = e.clientX;
        startY = e.clientY;
        
        // 获取元素的初始位置
        const transform = el.getAttribute('transform');
        if (transform) {
          const match = transform.match(/translate\(([^,]+),([^)]+)\)/);
          if (match) {
            initialX = parseFloat(match[1]);
            initialY = parseFloat(match[2]);
          }
        }
        
        // 触发dragstart回调
        if (dragHandler.onDragStart) {
          dragHandler.onDragStart.call(el, e, {
            x: initialX,
            y: initialY,
            dx: 0,
            dy: 0,
            sourceEvent: e
          });
        }
        
        document.addEventListener('mousemove', onMouseMove);
        document.addEventListener('mouseup', onMouseUp);
      };

      const onMouseMove = (e) => {
        if (!isDragging) return;
        
        const dx = e.clientX - startX;
        const dy = e.clientY - startY;
        const newX = initialX + dx;
        const newY = initialY + dy;
        
        // 触发drag回调
        if (dragHandler.onDrag) {
          dragHandler.onDrag.call(el, e, {
            x: newX,
            y: newY,
            dx: dx,
            dy: dy,
            sourceEvent: e
          });
        }
        
        // 默认行为：更新transform属性
        el.setAttribute('transform', `translate(${newX}, ${newY})`);
      };

      const onMouseUp = (e) => {
        isDragging = false;
        
        // 触发dragend回调
        if (dragHandler.onDragEnd) {
          dragHandler.onDragEnd.call(el, e, {
            x: initialX + (e.clientX - startX),
            y: initialY + (e.clientY - startY),
            dx: e.clientX - startX,
            dy: e.clientY - startY,
            sourceEvent: e
          });
        }
        
        document.removeEventListener('mousemove', onMouseMove);
        document.removeEventListener('mouseup', onMouseUp);
      };

      el.addEventListener('mousedown', onMouseDown);
      
      // 存储清理函数
      if (!this._dragCleanups) this._dragCleanups = new Map();
      this._dragCleanups.set(el, () => {
        el.removeEventListener('mousedown', onMouseDown);
        document.removeEventListener('mousemove', onMouseMove);
        document.removeEventListener('mouseup', onMouseUp);
      });
    });
    
    return this;
  }

  // 移除drag功能
  dragOff() {
    this._elements.forEach(el => {
      const cleanup = this._dragCleanups?.get(el);
      if (cleanup) {
        cleanup();
        this._dragCleanups.delete(el);
      }
    });
    return this;
  }

  // 添加子元素
  append(type) {
    const newElements = [];
    this._elements.forEach(parent => {
      let child;
      if (typeof type === 'function') {
        const tagName = type.call(parent, parent.__data__, this._elements.indexOf(parent));
        child = document.createElementNS('http://www.w3.org/2000/svg', tagName);
      } else {
        child = document.createElementNS('http://www.w3.org/2000/svg', type);
      }
      parent.appendChild(child);
      newElements.push(child);
    });
    return new D3Like(newElements);
  }

  // 插入元素（在指定元素前）
  insert(type, before) {
    const newElements = [];
    this._elements.forEach(parent => {
      let child;
      if (typeof type === 'function') {
        const tagName = type.call(parent, parent.__data__, this._elements.indexOf(parent));
        child = document.createElementNS('http://www.w3.org/2000/svg', tagName);
      } else {
        child = document.createElementNS('http://www.w3.org/2000/svg', type);
      }
      const refNode = typeof before === 'function' 
        ? before.call(parent, parent.__data__, this._elements.indexOf(parent))
        : parent.querySelector(before);
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

    const update = [];
    const exit = [];
    const enterData = [];

    if (keyFunction) {
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

      dataMap.forEach((d) => {
        enterData.push(d);
      });
    } else {
      this._elements.forEach((el, i) => {
        if (i < dataArray.length) {
          update.push(el);
          el.__data__ = dataArray[i];
        } else {
          exit.push(el);
        }
      });

      for (let i = this._elements.length; i < dataArray.length; i++) {
        enterData.push(dataArray[i]);
      }
    }

    // 创建进入选择集
    const enterSelection = {
      append: (type) => {
        const newElements = [];
        const parent = this._elements[0]?.parentNode;
        if (parent) {
          enterData.forEach((data, idx) => {
            const el = document.createElementNS('http://www.w3.org/2000/svg', type);
            el.__data__ = data;
            parent.appendChild(el);
            newElements.push(el);
          });
        }
        return new D3Like(newElements);
      },
      // 添加join方法支持
      join: (enterFn) => {
        if (enterFn) enterFn(enterSelection);
        return new D3Like(update);
      }
    };

    // 退出选择集
    const exitSelection = new D3Like(exit);
    
    return {
      enter: () => enterSelection,
      exit: () => exitSelection,
      join: (enterFn, updateFn, exitFn) => {
        if (enterFn) enterFn(enterSelection);
        if (updateFn) updateFn(new D3Like(update));
        if (exitFn) exitFn(exitSelection);
        return new D3Like(update);
      },
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
      if (typeof value === 'function') {
        el.__data__ = value.call(el, el.__data__, this._elements.indexOf(el));
      } else {
        el.__data__ = value;
      }
    });
    return this;
  }

  // 过渡动画
  transition(duration = 250, ease = 'ease') {
    const transitions = [];
    this._elements.forEach(el => {
      const transition = {
        _el: el,
        _duration: duration,
        _ease: ease,
        _props: [],
        
        attr: (name, value) => {
          const start = parseFloat(el.getAttribute(name)) || 0;
          const end = typeof value === 'function' 
            ? value.call(el, el.__data__, 0)
            : parseFloat(value);
          const startTime = performance.now();
          
          const animate = (currentTime) => {
            const elapsed = currentTime - startTime;
            const progress = Math.min(1, elapsed / this._duration);
            const eased = this._ease(progress, this._ease);
            const currentValue = start + (end - start) * eased;
            el.setAttribute(name, currentValue);
            if (progress < 1) requestAnimationFrame(animate);
          };
          requestAnimationFrame(animate);
          return transition;
        },
        
        style: (name, value) => {
          const start = parseFloat(el.style[name]) || 0;
          const end = typeof value === 'function'
            ? value.call(el, el.__data__, 0)
            : parseFloat(value);
          const startTime = performance.now();
          
          const animate = (currentTime) => {
            const elapsed = currentTime - startTime;
            const progress = Math.min(1, elapsed / this._duration);
            const eased = this._ease(progress, this._ease);
            const currentValue = start + (end - start) * eased;
            el.style[name] = currentValue;
            if (progress < 1) requestAnimationFrame(animate);
          };
          requestAnimationFrame(animate);
          return transition;
        },
        
        duration: (ms) => {
          this._duration = ms;
          return transition;
        },
        
        ease: (fn) => {
          this._ease = fn;
          return transition;
        },
        
        on: (event, handler) => {
          if (event === 'end' && handler) {
            // 简化处理，实际应该等待动画结束
            setTimeout(handler, this._duration);
          }
          return transition;
        }
      };
      transitions.push(transition);
    });
    return transitions[0] || {};
  }

  _ease(t, ease) {
    if (typeof ease === 'function') return ease(t);
    switch(ease) {
      case 'linear': return t;
      case 'ease-in': return t * t;
      case 'ease-out': return t * (2 - t);
      case 'ease-in-out': return t < 0.5 ? 4 * t * t * t : 1 - Math.pow(-2 * t + 2, 3) / 2;
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

  // 创建SVG
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
  // 辅助函数
  scaleLinear: () => ({
    domain: () => {},
    range: () => {},
    interpolate: () => {}
  }),
  max: (arr, fn) => Math.max(...arr.map(fn)),
  min: (arr, fn) => Math.min(...arr.map(fn)),
  // drag函数（全局）
  drag: () => {
    const dragObj = {
      _handlers: {},
      on: (event, handler) => {
        dragObj._handlers[event] = handler;
        return dragObj;
      },
      apply: (selection) => {
        selection.drag({
          onDragStart: dragObj._handlers.start,
          onDrag: dragObj._handlers.drag,
          onDragEnd: dragObj._handlers.end
        });
        return selection;
      }
    };
    return dragObj;
  }
};