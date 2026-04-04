# AllOSSystem 紧急修复指南

## ⚠️ 当前状态

原AllOSSystem项目源码已丢失，当前目录是v86库本身的代码。

## 🚀 快速修复步骤

由于之前已经在main分支提交了修复代码，请按以下步骤恢复并部署：

### 1. 克隆仓库（不含gh-pages）
```bash
cd /Users/zzx/WorkBuddy
rm -rf AllOSSystem
git clone https://github.com/13888285815/AllOSSystem.git --single-branch --branch main AllOSSystem
```

### 2. 检查修复是否存在
```bash
cd AllOSSystem
git log --oneline -5
# 应该看到：fix: 修复v86 CDN加载问题，使用unpkg作为主要源
```

### 3. 安装依赖并构建
```bash
npm install
npm run build
```

### 4. 部署到GitHub Pages（推荐方法）
```bash
# 创建干净的gh-pages分支
git checkout --orphan gh-pages

# 删除所有文件（保留.git）
git rm -rf .

# 复制dist内容
cp -r dist/* .

# 创建.gitignore排除不需要的文件
echo "node_modules" > .gitignore
echo ".DS_Store" >> .gitignore

# 提交
git add -A
git commit -m "deploy: AllOSSystem v1.0 - 修复CDN加载问题"

# 推送
git push -u origin gh-pages --force
```

### 5. 验证部署

等待1-2分钟后访问：
- https://13888285815.github.io/AllOSSystem/

应该看到：
- ✓ 页面正常加载
- ✓ 点击"进入系统展厅"
- ✓ 点击任意操作系统
- ✓ 点击"立即运行"按钮
- ✓ 控制台显示：`✓ v86 脚本加载成功，来源: https://unpkg.com/v86@3.0.2/libv86.js`

## 📋 修复内容总结

### 修改文件
- `src/pages/Runner.vue`

### 修改内容

#### 1. loadV86Script() 函数
- **修改前**：使用单一CDN源 `https://copy.sh/v86/build/libv86.js`
- **修改后**：使用3个CDN源自动切换
  1. `https://unpkg.com/v86@3.0.2/libv86.js` (主要)
  2. `https://cdn.jsdelivr.net/npm/v86@3.0.2/libv86.js` (备用1)
  3. `https://copy.sh/v86/build/libv86.js` (备用2)

#### 2. buildConfig() 函数
- **修改前**：wasm、bios等资源使用copy.sh
- **修改后**：统一使用unpkg作为CDN基础路径

#### 3. 超时设置
- **修改前**：15秒超时
- **修改后**：30秒超时

## 🔧 如果main分支没有修复

如果git log中没有看到修复提交，需要手动应用修复：

### 创建 src/pages/Runner.vue 修复

```javascript
// 找到 loadV86Script 函数，替换为：
const loadV86Script = () => {
  return new Promise((resolve, reject) => {
    if (window.V86) { resolve(); return }

    const timeout = setTimeout(() => {
      reject(new Error('v86 脚本加载超时'))
    }, 30000)

    const cdnSources = [
      'https://unpkg.com/v86@3.0.2/libv86.js',
      'https://cdn.jsdelivr.net/npm/v86@3.0.2/libv86.js',
      'https://copy.sh/v86/build/libv86.js'
    ]

    let currentIndex = 0

    const loadFromSource = (index) => {
      if (index >= cdnSources.length) {
        reject(new Error('所有 v86 CDN 源都加载失败，请检查网络连接'))
        return
      }

      const script = document.createElement('script')
      script.crossOrigin = 'anonymous'
      script.src = cdnSources[index]
      script.onload = () => {
        clearTimeout(timeout)
        console.log(`✓ v86 脚本加载成功，来源: ${cdnSources[index]}`)
        resolve()
      }
      script.onerror = () => {
        console.warn(`✗ v86 脚本加载失败，来源: ${cdnSources[index]}，尝试下一个...`)
        script.remove()
        loadFromSource(index + 1)
      }
      document.head.appendChild(script)
    }

    loadFromSource(currentIndex)
  })
}

// 找到 buildConfig 函数，修改为：
const buildConfig = () => {
  const cfg = os.value.config
  const cdnBase = 'https://unpkg.com/v86@3.0.2'
  const base = {
    wasm_path: cdnBase + '/v86.wasm',
    memory_size: (cfg.memory_size || 128) * 1024 * 1024,
    vga_memory_size: (cfg.vga_memory_size || 8) * 1024 * 1024,
    screen_container: v86Container.value,
    bios: { url: cdnBase + '/bios/seabios.bin' },
    vga_bios: { url: cdnBase + '/bios/vgabios.bin' },
    autostart: true,
    disable_mouse: false,
    acpi: cfg.acpi === true,
    audio: false,
    boot_order: cfg.boot_order || 0x132,
  }
  // ... 保持其他代码不变
}
```

## 📞 遇到问题？

如果部署后仍然无法运行，请检查：

1. **GitHub Pages状态**
   - 访问：https://github.com/13888285815/AllOSSystem/settings/pages
   - 确认Source是gh-pages分支

2. **控制台错误**
   - F12打开开发者工具
   - 查看Console标签是否有错误
   - 查看Network标签检查v86资源是否加载成功

3. **CDN可访问性**
   - 访问：https://unpkg.com/v86@3.0.2/libv86.js
   - 应该看到JS代码

4. **网络连接**
   - 确保能访问GitHub Pages
   - 确保能访问unpkg.com

## 💡 后续优化建议

1. **添加GitHub Actions自动部署**
2. **本地缓存v86资源**，减少CDN依赖
3. **添加离线模式**支持
4. **优化镜像加载速度**，使用懒加载
