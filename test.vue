/**
 * 移除元素上的所有事件监听器
 * @param {HTMLElement} element - 目标元素
 * @param {boolean} removeChildren - 是否同时处理子元素
 */
function removeAllEventListeners(element, removeChildren = false) {
    // 1. 克隆并替换当前元素（移除所有事件）
    const clone = element.cloneNode(true);
    element.parentNode.replaceChild(clone, element);
    
    // 2. 如果需要，递归处理子元素
    if (removeChildren) {
        const allElements = clone.getElementsByTagName('*');
        for (let i = 0; i < allElements.length; i++) {
            const childClone = allElements[i].cloneNode(true);
            allElements[i].parentNode.replaceChild(childClone, allElements[i]);
        }
    }
    
    return clone;
}

// 使用示例
const button = document.getElementById('myButton');
const cleanButton = removeAllEventListeners(button, true);