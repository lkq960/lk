"""
主应用类
"""
import tkinter as tk
from tkinter import ttk, messagebox
import threading
import time
import logging
from typing import Optional

from src.core.window_manager import WindowManager, WindowSettings, WindowState
from src.core.mouse_tracker import MouseTracker
from src.utils.config import ConfigManager

logger = logging.getLogger(__name__)

class FloatingApp:
    """悬浮窗主应用"""
    
    def __init__(self):
        self.config = ConfigManager()
        self.window_manager: Optional[WindowManager] = None
        self.mouse_tracker: Optional[MouseTracker] = None
        self.ui_components = {}
        self.is_running = False
        
        # 初始化组件
        self._init_components()
    
    def _init_components(self):
        """初始化组件"""
        try:
            # 加载配置
            window_config = self.config.get("window", {})
            
            # 创建窗口设置
            window_settings = WindowSettings(
                title="悬浮窗工具箱",
                width=window_config.get("width", 320),
                height=window_config.get("height", 240),
                always_on_top=window_config.get("always_on_top", True),
                alpha=window_config.get("transparency", 0.9),
                borderless=window_config.get("borderless", True),
                follow_mouse=window_config.get("follow_mouse", True),
                follow_offset_x=window_config.get("follow_offset_x", 20),
                follow_offset_y=window_config.get("follow_offset_y", 20)
            )
            
            # 创建窗口管理器
            self.window_manager = WindowManager(window_settings)
            self.window_manager.create_window()
            
            # 创建鼠标跟踪器
            update_interval = self.config.get("performance.update_interval_ms", 50) / 1000.0
            self.mouse_tracker = MouseTracker(update_interval=update_interval)
            
            # 设置回调
            if self.window_manager.root:
                self.mouse_tracker.add_callback(self._on_mouse_move)
                self.window_manager.add_close_callback(self._on_window_close)
            
            logger.info("组件初始化完成")
            
        except Exception as e:
            logger.error(f"组件初始化失败: {e}")
            raise
    
    def _create_ui(self):
        """创建用户界面"""
        if not self.window_manager or not self.window_manager.root:
            return
        
        root = self.window_manager.root
        
        try:
            # 创建主框架
            main_frame = ttk.Frame(root)
            main_frame.pack(fill='both', expand=True, padx=5, pady=5)
            self.ui_components['main_frame'] = main_frame
            
            # 标题栏
            title_frame = ttk.Frame(main_frame)
            title_frame.pack(fill='x', pady=(0, 10))
            
            title_label = ttk.Label(
                title_frame, 
                text="悬浮窗工具箱", 
                font=('Microsoft YaHei', 12, 'bold')
            )
            title_label.pack(side='left')
            
            # 控制按钮
            control_frame = ttk.Frame(title_frame)
            control_frame.pack(side='right')
            
            # 最小化按钮
            min_btn = ttk.Button(
                control_frame,
                text="－",
                width=3,
                command=self._minimize_window
            )
            min_btn.pack(side='left', padx=2)
            
            # 关闭按钮
            close_btn = ttk.Button(
                control_frame,
                text="×",
                width=3,
                command=self._on_window_close
            )
            close_btn.pack(side='left', padx=2)
            
            # 内容区域
            content_frame = ttk.Frame(main_frame)
            content_frame.pack(fill='both', expand=True)
            
            # 状态信息
            self.status_label = ttk.Label(
                content_frame,
                text="准备就绪",
                justify='left'
            )
            self.status_label.pack(fill='x', pady=5)
            
            # 鼠标位置显示
            self.mouse_pos_label = ttk.Label(
                content_frame,
                text="鼠标位置: (0, 0)",
                justify='left'
            )
            self.mouse_pos_label.pack(fill='x', pady=2)
            
            # 控制按钮区域
            button_frame = ttk.Frame(content_frame)
            button_frame.pack(fill='x', pady=10)
            
            # 跟随按钮
            self.follow_btn = ttk.Button(
                button_frame,
                text="暂停跟随",
                command=self._toggle_follow
            )
            self.follow_btn.pack(side='left', padx=5)
            
            # 设置按钮
            settings_btn = ttk.Button(
                button_frame,
                text="设置",
                command=self._open_settings
            )
            settings_btn.pack(side='left', padx=5)
            
            # 关于按钮
            about_btn = ttk.Button(
                button_frame,
                text="关于",
                command=self._show_about
            )
            about_btn.pack(side='left', padx=5)
            
            logger.info("用户界面创建完成")
            
        except Exception as e:
            logger.error(f"创建用户界面失败: {e}")
    
    def _on_mouse_move(self, x: int, y: int, timestamp: float):
        """鼠标移动回调"""
        try:
            # 更新鼠标位置显示
            if hasattr(self, 'mouse_pos_label'):
                self.mouse_pos_label.config(text=f"鼠标位置: ({x}, {y})")
            
            # 窗口跟随
            if self.window_manager:
                self.window_manager.follow_mouse(x, y)
            
            # 更新状态
            if hasattr(self, 'status_label'):
                self.status_label.config(text=f"运行中 - 跟随模式")
                
        except Exception as e:
            logger.error(f"鼠标移动回调处理失败: {e}")
    
    def _toggle_follow(self):
        """切换跟随模式"""
        if self.window_manager:
            self.window_manager.settings.follow_mouse = not self.window_manager.settings.follow_mouse
            
            # 更新按钮文字
            if self.window_manager.settings.follow_mouse:
                self.follow_btn.config(text="暂停跟随")
                self.status_label.config(text="已启用鼠标跟随")
            else:
                self.follow_btn.config(text="开始跟随")
                self.status_label.config(text="已暂停鼠标跟随")
            
            # 保存配置
            self.config.set("window.follow_mouse", self.window_manager.settings.follow_mouse)
            self.config.save()
    
    def _minimize_window(self):
        """最小化窗口"""
        if self.window_manager:
            self.window_manager.hide()
    
    def _open_settings(self):
        """打开设置窗口"""
        # 这里可以打开设置对话框
        messagebox.showinfo("设置", "设置功能正在开发中...")
    
    def _show_about(self):
        """显示关于对话框"""
        about_text = """
        悬浮窗工具箱 v1.0.0
        
        功能特性：
        • 实时鼠标跟随
        • 屏幕信息显示
        • 文字识别（OCR）
        • 颜色拾取器
        
        作者：您的名字
        许可证：MIT
        
        感谢使用！
        """
        messagebox.showinfo("关于", about_text)
    
    def _on_window_close(self):
        """窗口关闭事件"""
        logger.info("正在关闭应用...")
        self.stop()
    
    def start(self):
        """启动应用"""
        try:
            # 创建UI
            self._create_ui()
            
            # 启动鼠标跟踪
            if self.mouse_tracker:
                self.mouse_tracker.start()
            
            # 显示窗口
            if self.window_manager:
                self.window_manager.show()
            
            self.is_running = True
            logger.info("应用启动成功")
            
        except Exception as e:
            logger.error(f"应用启动失败: {e}")
            raise
    
    def stop(self):
        """停止应用"""
        try:
            self.is_running = False
            
            # 停止鼠标跟踪
            if self.mouse_tracker:
                self.mouse_tracker.stop()
            
            # 关闭窗口
            if self.window_manager and self.window_manager.root:
                self.window_manager.root.quit()
                self.window_manager.root.destroy()
            
            # 保存配置
            self.config.save()
            
            logger.info("应用已停止")
            
        except Exception as e:
            logger.error(f"停止应用时出错: {e}")
    
    def run(self):
        """运行应用主循环"""
        try:
            self.start()
            
            if self.window_manager:
                self.window_manager.run()
            else:
                logger.error("窗口管理器未初始化")
                
        except KeyboardInterrupt:
            logger.info("收到中断信号")
            self.stop()
        except Exception as e:
            logger.error(f"应用运行出错: {e}")
            self.stop()
