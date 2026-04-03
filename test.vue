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
    if (typeof fn === 'function') {
      fn(this, ...args);
    } else {
      console.warn('call() 的第一个参数必须是函数');
    }
    return this;
  }

  // 属性操作
  attr(name, value) {
    if (value === undefined) {
      return this._elements[0]?.getAttribute(name);
    }
    this._elements.forEach((el, i) => {
      if (typeof value === 'function') {
        const val = value.call(el, el.__data__, i);
        if (val === null) {
          el.removeAttribute(name);
        } else {
          el.setAttribute(name, val);
        }
      } else if (value === null) {
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
    this._elements.forEach((el, i) => {
      if (typeof value === 'function') {
        const val = value.call(el, el.__data__, i);
        el.style.setProperty(name, val, priority || '');
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
    this._elements.forEach((el, i) => {
      if (typeof content === 'function') {
        el.textContent = content.call(el, el.__data__, i);
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
    this._elements.forEach((el, i) => {
      if (typeof content === 'function') {
        el.innerHTML = content.call(el, el.__data__, i);
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
    this._elements.forEach((el, i) => {
      const shouldAdd = typeof value === 'function' 
        ? value.call(el, el.__data__, i)
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
        if (stored && stored[event]) {
          el.removeEventListener(event, stored[event]);
          delete stored[event];
        }
      } else {
        if (!this._eventHandlers) this._eventHandlers = new Map();
        let elHandlers = this._eventHandlers.get(el);
        if (!elHandlers) {
          elHandlers = {};
          this._eventHandlers.set(el, elHandlers);
        }
        // 移除旧的监听器
        if (elHandlers[event]) {
          el.removeEventListener(event, elHandlers[event]);
        }
        elHandlers[event] = handler;
        el.addEventListener(event, handler, options);
      }
    });
    return this;
  }

  // drag方法：实现拖拽功能
  drag(dragHandler) {
    this._elements.forEach(el => {
      let startX = 0, startY = 0;
      let initialX = 0, initialY = 0;
      let initialTransform = { x: 0, y: 0 };
      let isDragging = false;

      const onMouseDown = (e) => {
        // 避免干扰文本选择等
        e.preventDefault();
        e.stopPropagation();
        
        isDragging = true;
        startX = e.clientX;
        startY = e.clientY;
        
        // 获取元素的初始位置（支持多种存储方式）
        const transform = el.getAttribute('transform');
        if (transform) {
          const translateMatch = transform.match(/translate\(\s*([^,\s)]+)\s*,\s*([^)\s]+)\s*\)/);
          if (translateMatch) {
            initialTransform.x = parseFloat(translateMatch[1]) || 0;
            initialTransform.y = parseFloat(translateMatch[2]) || 0;
          }
        }
        
        // 也支持cx,cy或x,y属性
        initialTransform.x = parseFloat(el.getAttribute('cx')) || 
                            parseFloat(el.getAttribute('x')) || 
                            initialTransform.x;
        initialTransform.y = parseFloat(el.getAttribute('cy')) || 
                            parseFloat(el.getAttribute('y')) || 
                            initialTransform.y;
        
        initialX = initialTransform.x;
        initialY = initialTransform.y;
        
        // 触发dragstart回调
        if (dragHandler && typeof dragHandler.onDragStart === 'function') {
          dragHandler.onDragStart.call(el, e, {
            x: initialX,
            y: initialY,
            dx: 0,
            dy: 0,
            sourceEvent: e,
            subject: el.__data__
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
        if (dragHandler && typeof dragHandler.onDrag === 'function') {
          dragHandler.onDrag.call(el, e, {
            x: newX,
            y: newY,
            dx: dx,
            dy: dy,
            sourceEvent: e,
            subject: el.__data__
          });
        }
        
        // 默认行为：更新transform或坐标属性
        if (el.getAttribute('transform') !== null || el.tagName === 'g') {
          el.setAttribute('transform', `translate(${newX}, ${newY})`);
        } else if (el.hasAttribute('cx')) {
          el.setAttribute('cx', newX);
          el.setAttribute('cy', newY);
        } else if (el.hasAttribute('x')) {
          el.setAttribute('x', newX);
          el.setAttribute('y', newY);
        }
      };

      const onMouseUp = (e) => {
        if (!isDragging) return;
        
        isDragging = false;
        const dx = e.clientX - startX;
        const dy = e.clientY - startY;
        
        // 触发dragend回调
        if (dragHandler && typeof dragHandler.onDragEnd === 'function') {
          dragHandler.onDragEnd.call(el, e, {
            x: initialX + dx,
            y: initialY + dy,
            dx: dx,
            dy: dy,
            sourceEvent: e,
            subject: el.__data__
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
    this._elements.forEach((parent, i) => {
      let child;
      if (typeof type === 'function') {
        const tagName = type.call(parent, parent.__data__, i);
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
    this._elements.forEach((parent, i) => {
      let child;
      if (typeof type === 'function') {
        const tagName = type.call(parent, parent.__data__, i);
        child = document.createElementNS('http://www.w3.org/2000/svg', tagName);
      } else {
        child = document.createElementNS('http://www.w3.org/2000/svg', type);
      }
      
      let refNode;
      if (typeof before === 'function') {
        const selector = before.call(parent, parent.__data__, i);
        refNode = parent.querySelector(selector);
      } else if (typeof before === 'string') {
        refNode = parent.querySelector(before);
      } else {
        refNode = before;
      }
      
      if (refNode) {
        parent.insertBefore(child, refNode);
      } else {
        parent.appendChild(child);
      }
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

    if (keyFunction && typeof keyFunction === 'function') {
      const dataMap = new Map();
      dataArray.forEach((d, i) => {
        const key = keyFunction(d, i);
        dataMap.set(key, d);
      });

      this._elements.forEach(el => {
        const elKey = el.__data__ !== undefined ? keyFunction(el.__data__, 0) : null;
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
    const parentNode = this._elements[0]?.parentNode;
    
    const enterSelection = {
      append: (type) => {
        const newElements = [];
        if (parentNode) {
          enterData.forEach((data) => {
            const el = document.createElementNS('http://www.w3.org/2000/svg', type);
            el.__data__ = data;
            parentNode.appendChild(el);
            newElements.push(el);
          });
        }
        return new D3Like(newElements);
      },
      appendToParent: (parent) => {
        const newElements = [];
        enterData.forEach((data) => {
          const el = document.createElementNS('http://www.w3.org/2000/svg', 'g');
          el.__data__ = data;
          parent.appendChild(el);
          newElements.push(el);
        });
        return new D3Like(newElements);
      }
    };

    // 退出选择集
    const exitSelection = new D3Like(exit);
    
    const result = {
      enter: () => enterSelection,
      exit: () => exitSelection,
      join: (enterFn, updateFn, exitFn) => {
        if (enterFn && typeof enterFn === 'function') enterFn(enterSelection);
        if (updateFn && typeof updateFn === 'function') updateFn(new D3Like(update));
        if (exitFn && typeof exitFn === 'function') exitFn(exitSelection);
        return new D3Like(update);
      }
    };
    
    // 添加便捷的enter().append()链式调用支持
    if (enterData.length > 0 && update.length === 0 && exit.length === 0) {
      return enterSelection;
    }
    
    return result;
  }

  // 获取/设置数据
  datum(value) {
    if (value === undefined) {
      return this._elements[0]?.__data__;
    }
    this._elements.forEach((el, i) => {
      if (typeof value === 'function') {
        el.__data__ = value.call(el, el.__data__, i);
      } else {
        el.__data__ = value;
      }
    });
    return this;
  }

  // 过渡动画
  transition(duration = 250, ease = 'ease') {
    const self = this;
    const transitions = [];
    
    this._elements.forEach(el => {
      const transitionObj = {
        _el: el,
        _duration: duration,
        _ease: ease,
        _animations: [],
        
        attr: function(name, value) {
          const start = parseFloat(el.getAttribute(name)) || 0;
          const end = typeof value === 'function' 
            ? value.call(el, el.__data__, 0)
            : parseFloat(value);
          const startTime = performance.now();
          
          const animate = (currentTime) => {
            const elapsed = currentTime - startTime;
            const progress = Math.min(1, elapsed / this._duration);
            const eased = self._ease(progress, this._ease);
            const currentValue = start + (end - start) * eased;
            el.setAttribute(name, currentValue);
            if (progress < 1) {
              requestAnimationFrame(animate);
            }
          }.bind(this);
          
          requestAnimationFrame(animate);
          return this;
        },
        
        style: function(name, value) {
          const start = parseFloat(el.style[name]) || 0;
          const end = typeof value === 'function'
            ? value.call(el, el.__data__, 0)
            : parseFloat(value);
          const startTime = performance.now();
          
          const animate = (currentTime) => {
            const elapsed = currentTime - startTime;
            const progress = Math.min(1, elapsed / this._duration);
            const eased = self._ease(progress, this._ease);
            const currentValue = start + (end - start) * eased;
            el.style[name] = currentValue;
            if (progress < 1) {
              requestAnimationFrame(animate);
            }
          }.bind(this);
          
          requestAnimationFrame(animate);
          return this;
        },
        
        duration: function(ms) {
          this._duration = ms;
          return this;
        },
        
        ease: function(fn) {
          this._ease = fn;
          return this;
        },
        
        on: function(event, handler) {
          if (event === 'end' && handler && typeof handler === 'function') {
            setTimeout(handler, this._duration);
          }
          return this;
        }
      };
      
      transitions.push(transitionObj);
    });
    
    return transitions[0] || this;
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
    if (typeof callback === 'function') {
      this._elements.forEach((el, i) => {
        callback.call(el, el.__data__, i);
      });
    }
    return this;
  }

  // 过滤选择集
  filter(selector) {
    const filtered = this._elements.filter((el, i) => {
      if (typeof selector === 'function') {
        return selector.call(el, el.__data__, i);
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
    domain: (d) => d,
    range: (r) => r,
    interpolate: () => {}
  }),
  max: (arr, fn) => {
    if (!arr || arr.length === 0) return undefined;
    if (!fn) return Math.max(...arr);
    return Math.max(...arr.map(fn));
  },
  min: (arr, fn) => {
    if (!arr || arr.length === 0) return undefined;
    if (!fn) return Math.min(...arr);
    return Math.min(...arr.map(fn));
  },
  // drag函数（全局）
  drag: () => {
    const dragObj = {
      _handlers: {},
      on: function(event, handler) {
        this._handlers[event] = handler;
        return this;
      },
      apply: function(selection) {
        if (selection && selection.drag) {
          selection.drag({
            onDragStart: this._handlers.start,
            onDrag: this._handlers.drag,
            onDragEnd: this._handlers.end
          });
        }
        return selection;
      }
    };
    return dragObj;
  }
};

// 使用示例和测试
if (typeof module !== 'undefined' && module.exports) {
  module.exports = { D3Like, d3: window.d3 };
}